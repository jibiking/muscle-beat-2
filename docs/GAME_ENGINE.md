# ゲームエンジン設計書

## 目次
1. [概要](#概要)
2. [ゲームループ設計](#ゲームループ設計)
3. [ノーツ判定ロジック](#ノーツ判定ロジック)
4. [モーションセンサー連携](#モーションセンサー連携)
5. [スコアリングシステム](#スコアリングシステム)
6. [ビートマップJSONスキーマ](#ビートマップjsonスキーマ)
7. [音楽同期メカニズム](#音楽同期メカニズム)
8. [ステートマシン](#ステートマシン)
9. [パフォーマンス最適化](#パフォーマンス最適化)

---

## 概要

本ドキュメントでは、筋トレ×音ゲー「Muscle Beat」のゲームエンジンの詳細設計を定義します。

### 技術スタック
- **React Native (Expo SDK 52+)**
- **TypeScript**
- **react-native-reanimated** (v3): アニメーション、ゲームループ
- **expo-av**: 音楽再生、同期
- **expo-sensors**: 加速度センサー

### 設計方針
- 60FPS維持を最優先
- UIスレッドとワークレットスレッドの適切な分離
- 型安全性の確保
- テスタビリティの考慮

---

## ゲームループ設計

### アーキテクチャ

ゲームループは**react-native-reanimated**の`useFrameCallback`を使用し、UIスレッドをブロックせずに高頻度更新を実現します。

```typescript
import { useFrameCallback, useSharedValue } from 'react-native-reanimated';

/**
 * ゲームループのコアフック
 * 60FPSでゲーム状態を更新
 */
export const useGameLoop = (
  isPlaying: boolean,
  onFrame: (deltaTime: number, currentTime: number) => void
) => {
  const lastTimestamp = useSharedValue<number>(0);

  useFrameCallback((frameInfo) => {
    'worklet';

    if (!isPlaying) {
      lastTimestamp.value = 0;
      return;
    }

    const currentTime = frameInfo.timestamp;

    if (lastTimestamp.value === 0) {
      lastTimestamp.value = currentTime;
      return;
    }

    const deltaTime = currentTime - lastTimestamp.value;
    lastTimestamp.value = currentTime;

    // デルタタイムをミリ秒に変換（frameInfo.timestampはミリ秒単位）
    onFrame(deltaTime, currentTime);
  }, isPlaying);
};
```

### フレーム処理フロー

```typescript
/**
 * ゲームエンジンのメインクラス
 */
export class GameEngine {
  private currentTimeMs = 0;
  private audioOffsetMs = 0;
  private notes: Note[] = [];
  private activeNotes: Note[] = [];

  /**
   * 毎フレーム呼び出される処理
   * @param deltaTime フレーム間の経過時間（ミリ秒）
   * @param audioPosition 音楽の現在位置（ミリ秒）
   */
  public onFrame(deltaTime: number, audioPosition: number): void {
    'worklet';

    // 1. 音楽位置との同期
    this.currentTimeMs = audioPosition + this.audioOffsetMs;

    // 2. アクティブノーツの更新
    this.updateActiveNotes();

    // 3. ノーツ判定ウィンドウのチェック
    this.checkJudgmentWindow();

    // 4. ミスノーツの処理
    this.processMissedNotes();

    // 5. UIへの状態通知（runOnJS経由）
    this.notifyFrameUpdate();
  }

  private updateActiveNotes(): void {
    'worklet';

    // 画面に表示すべきノーツをフィルタリング
    // 判定ライン到達の2秒前から表示
    const DISPLAY_AHEAD_MS = 2000;
    const displayWindowStart = this.currentTimeMs;
    const displayWindowEnd = this.currentTimeMs + DISPLAY_AHEAD_MS;

    this.activeNotes = this.notes.filter(note =>
      !note.judged &&
      note.targetTime >= displayWindowStart &&
      note.targetTime <= displayWindowEnd
    );
  }
}
```

### 実装例：ゲームコンポーネント

```typescript
import React, { useCallback } from 'react';
import { runOnJS, useSharedValue } from 'react-native-reanimated';

export const GameScreen: React.FC = () => {
  const [gameState, setGameState] = useState<GameState>('READY');
  const gameEngine = useRef(new GameEngine()).current;
  const audioPosition = useSharedValue(0);

  // ワークレットから呼び出される更新ハンドラ
  const handleFrameUpdate = useCallback((
    activeNotes: Note[],
    score: number,
    combo: number
  ) => {
    // UIスレッドで状態更新
    setActiveNotes(activeNotes);
    setScore(score);
    setCombo(combo);
  }, []);

  useGameLoop(
    gameState === 'PLAYING',
    useCallback((deltaTime: number, currentTime: number) => {
      'worklet';

      // 音楽位置を取得（SharedValue）
      const audioPos = audioPosition.value;

      // ゲームエンジン更新
      gameEngine.onFrame(deltaTime, audioPos);

      // UI更新（runOnJS経由）
      runOnJS(handleFrameUpdate)(
        gameEngine.getActiveNotes(),
        gameEngine.getScore(),
        gameEngine.getCombo()
      );
    }, [gameEngine, handleFrameUpdate])
  );

  // ... レンダリング
};
```

---

## ノーツ判定ロジック

### 判定ウィンドウ定義

```typescript
/**
 * 判定の種類
 */
export enum JudgmentType {
  PERFECT = 'PERFECT',
  GREAT = 'GREAT',
  GOOD = 'GOOD',
  MISS = 'MISS',
}

/**
 * 判定ウィンドウ設定（ミリ秒）
 */
export const JUDGMENT_WINDOWS = {
  [JudgmentType.PERFECT]: 50,   // ±50ms
  [JudgmentType.GREAT]: 120,     // ±120ms
  [JudgmentType.GOOD]: 265,      // ±265ms
  [JudgmentType.MISS]: 310,      // ±310ms（この時間を超えたらMiss確定）
} as const;

/**
 * 判定結果
 */
export interface JudgmentResult {
  type: JudgmentType;
  timing: number;          // 判定タイミング（ms）
  deviation: number;       // 目標時間からのズレ（ms、早い場合は負、遅い場合は正）
  note: Note;
}
```

### 判定アルゴリズム

```typescript
export class JudgmentSystem {
  /**
   * ノーツ判定を実行
   * @param note 判定対象のノーツ
   * @param inputTime 入力タイミング（ms）
   * @returns 判定結果
   */
  public judge(note: Note, inputTime: number): JudgmentResult {
    'worklet';

    const deviation = inputTime - note.targetTime;
    const absDeviation = Math.abs(deviation);

    let judgmentType: JudgmentType;

    if (absDeviation <= JUDGMENT_WINDOWS[JudgmentType.PERFECT]) {
      judgmentType = JudgmentType.PERFECT;
    } else if (absDeviation <= JUDGMENT_WINDOWS[JudgmentType.GREAT]) {
      judgmentType = JudgmentType.GREAT;
    } else if (absDeviation <= JUDGMENT_WINDOWS[JudgmentType.GOOD]) {
      judgmentType = JudgmentType.GOOD;
    } else {
      judgmentType = JudgmentType.MISS;
    }

    return {
      type: judgmentType,
      timing: inputTime,
      deviation,
      note,
    };
  }

  /**
   * ノーツがMiss判定確定になったかチェック
   * @param note チェック対象のノーツ
   * @param currentTime 現在時刻（ms）
   * @returns Miss確定ならtrue
   */
  public isMissConfirmed(note: Note, currentTime: number): boolean {
    'worklet';

    const deviation = currentTime - note.targetTime;
    return deviation > JUDGMENT_WINDOWS[JudgmentType.MISS];
  }

  /**
   * ノーツが判定可能ウィンドウ内にあるかチェック
   * @param note チェック対象のノーツ
   * @param currentTime 現在時刻（ms）
   * @returns 判定可能ならtrue
   */
  public isInJudgmentWindow(note: Note, currentTime: number): boolean {
    'worklet';

    const deviation = Math.abs(currentTime - note.targetTime);
    return deviation <= JUDGMENT_WINDOWS[JudgmentType.MISS];
  }
}
```

### 先行入力・遅延入力の扱い

```typescript
export class InputBuffer {
  private bufferedInputs: Array<{ time: number; exerciseType: ExerciseType }> = [];
  private readonly BUFFER_DURATION_MS = 100; // 100ms分のバッファ

  /**
   * 入力をバッファに追加
   */
  public addInput(time: number, exerciseType: ExerciseType): void {
    'worklet';

    this.bufferedInputs.push({ time, exerciseType });

    // 古い入力を削除
    this.bufferedInputs = this.bufferedInputs.filter(
      input => time - input.time <= this.BUFFER_DURATION_MS
    );
  }

  /**
   * 指定時刻付近の入力を検索
   */
  public findMatchingInput(
    targetTime: number,
    exerciseType: ExerciseType,
    windowMs: number
  ): { time: number } | null {
    'worklet';

    // 最も近い入力を探す
    let closestInput: { time: number } | null = null;
    let minDeviation = Infinity;

    for (const input of this.bufferedInputs) {
      if (input.exerciseType !== exerciseType) continue;

      const deviation = Math.abs(input.time - targetTime);
      if (deviation <= windowMs && deviation < minDeviation) {
        closestInput = input;
        minDeviation = deviation;
      }
    }

    return closestInput;
  }

  /**
   * バッファをクリア
   */
  public clear(): void {
    'worklet';
    this.bufferedInputs = [];
  }
}
```

---

## モーションセンサー連携

### センサーデータ型定義

```typescript
import { Accelerometer, AccelerometerMeasurement } from 'expo-sensors';

/**
 * 運動種目
 */
export enum ExerciseType {
  SQUAT = 'SQUAT',        // スクワット
  SIT_UP = 'SIT_UP',      // 腹筋
  BURPEE = 'BURPEE',      // バーピー
}

/**
 * センサーデータのサンプル
 */
export interface SensorSample {
  timestamp: number;      // サンプル取得時刻（ms）
  x: number;              // X軸加速度（m/s²）
  y: number;              // Y軸加速度（m/s²）
  z: number;              // Z軸加速度（m/s²）
}

/**
 * 運動検知結果
 */
export interface ExerciseDetection {
  type: ExerciseType;
  timestamp: number;      // 検知時刻（ms）
  confidence: number;     // 信頼度（0.0～1.0）
}
```

### 種目別検知アルゴリズム

```typescript
/**
 * 各運動種目の検知しきい値
 */
const EXERCISE_THRESHOLDS = {
  [ExerciseType.SQUAT]: {
    minYAcceleration: 1.5,      // Y軸最小加速度変化（m/s²）
    minPeakInterval: 500,       // 最小ピーク間隔（ms）
    maxPeakInterval: 2000,      // 最大ピーク間隔（ms）
  },
  [ExerciseType.SIT_UP]: {
    minZTiltChange: 0.8,        // Z軸最小傾き変化（m/s²）
    minDuration: 400,           // 最小動作時間（ms）
  },
  [ExerciseType.BURPEE]: {
    minYAcceleration: 2.5,      // Y軸最小加速度変化（m/s²、大きめ）
    requiredPhases: 2,          // 必要な動作フェーズ数（しゃがむ→立つ）
    maxPhaseInterval: 1000,     // 最大フェーズ間隔（ms）
  },
} as const;

/**
 * モーションセンサーマネージャー
 */
export class MotionSensorManager {
  private subscription: any = null;
  private sensorData: SensorSample[] = [];
  private lastDetection: { [key in ExerciseType]?: number } = {};

  private readonly SAMPLING_RATE = 60;              // 60Hz
  private readonly COOLDOWN_MS = 300;               // クールダウン時間
  private readonly HISTORY_DURATION_MS = 2000;      // 履歴保持時間

  /**
   * センサー監視を開始
   */
  public start(onDetection: (detection: ExerciseDetection) => void): void {
    Accelerometer.setUpdateInterval(1000 / this.SAMPLING_RATE);

    this.subscription = Accelerometer.addListener((data: AccelerometerMeasurement) => {
      const sample: SensorSample = {
        timestamp: Date.now(),
        x: data.x,
        y: data.y,
        z: data.z,
      };

      this.addSample(sample);

      // 運動検知を試行
      const detection = this.detectExercise();
      if (detection) {
        onDetection(detection);
      }
    });
  }

  /**
   * センサー監視を停止
   */
  public stop(): void {
    if (this.subscription) {
      this.subscription.remove();
      this.subscription = null;
    }
    this.sensorData = [];
  }

  /**
   * サンプルを履歴に追加
   */
  private addSample(sample: SensorSample): void {
    this.sensorData.push(sample);

    // 古いデータを削除
    const cutoffTime = sample.timestamp - this.HISTORY_DURATION_MS;
    this.sensorData = this.sensorData.filter(s => s.timestamp >= cutoffTime);
  }

  /**
   * 運動検知を実行
   */
  private detectExercise(): ExerciseDetection | null {
    const now = Date.now();

    // スクワット検知
    const squatDetection = this.detectSquat(now);
    if (squatDetection) return squatDetection;

    // 腹筋検知
    const sitUpDetection = this.detectSitUp(now);
    if (sitUpDetection) return sitUpDetection;

    // バーピー検知
    const burpeeDetection = this.detectBurpee(now);
    if (burpeeDetection) return burpeeDetection;

    return null;
  }

  /**
   * スクワット検知
   * Y軸加速度の周期的な変化を検出
   */
  private detectSquat(now: number): ExerciseDetection | null {
    // クールダウンチェック
    if (this.isInCooldown(ExerciseType.SQUAT, now)) {
      return null;
    }

    const threshold = EXERCISE_THRESHOLDS[ExerciseType.SQUAT];

    // Y軸のピークを検出
    const peaks = this.findPeaks(
      this.sensorData.map(s => ({ time: s.timestamp, value: s.y })),
      threshold.minYAcceleration
    );

    if (peaks.length < 2) return null;

    // 最新の2つのピーク間隔をチェック
    const latestPeak = peaks[peaks.length - 1];
    const previousPeak = peaks[peaks.length - 2];
    const interval = latestPeak.time - previousPeak.time;

    if (interval >= threshold.minPeakInterval && interval <= threshold.maxPeakInterval) {
      this.lastDetection[ExerciseType.SQUAT] = now;
      return {
        type: ExerciseType.SQUAT,
        timestamp: now,
        confidence: 0.9,
      };
    }

    return null;
  }

  /**
   * 腹筋検知
   * Z軸の傾き変化を検出
   */
  private detectSitUp(now: number): ExerciseDetection | null {
    if (this.isInCooldown(ExerciseType.SIT_UP, now)) {
      return null;
    }

    const threshold = EXERCISE_THRESHOLDS[ExerciseType.SIT_UP];

    if (this.sensorData.length < 10) return null;

    // 直近1秒間のZ軸変化を計算
    const recentData = this.sensorData.filter(
      s => now - s.timestamp <= 1000
    );

    if (recentData.length < 5) return null;

    const zValues = recentData.map(s => s.z);
    const minZ = Math.min(...zValues);
    const maxZ = Math.max(...zValues);
    const zChange = maxZ - minZ;

    if (zChange >= threshold.minZTiltChange) {
      this.lastDetection[ExerciseType.SIT_UP] = now;
      return {
        type: ExerciseType.SIT_UP,
        timestamp: now,
        confidence: 0.85,
      };
    }

    return null;
  }

  /**
   * バーピー検知
   * 大きなY軸加速度変化（2フェーズ）を検出
   */
  private detectBurpee(now: number): ExerciseDetection | null {
    if (this.isInCooldown(ExerciseType.BURPEE, now)) {
      return null;
    }

    const threshold = EXERCISE_THRESHOLDS[ExerciseType.BURPEE];

    // Y軸の大きなピークを検出
    const peaks = this.findPeaks(
      this.sensorData.map(s => ({ time: s.timestamp, value: s.y })),
      threshold.minYAcceleration
    );

    if (peaks.length < threshold.requiredPhases) return null;

    // 最新の2つのピークをチェック
    const latestPeak = peaks[peaks.length - 1];
    const previousPeak = peaks[peaks.length - 2];
    const interval = latestPeak.time - previousPeak.time;

    if (interval <= threshold.maxPhaseInterval) {
      this.lastDetection[ExerciseType.BURPEE] = now;
      return {
        type: ExerciseType.BURPEE,
        timestamp: now,
        confidence: 0.88,
      };
    }

    return null;
  }

  /**
   * クールダウン期間中かチェック
   */
  private isInCooldown(exerciseType: ExerciseType, now: number): boolean {
    const lastTime = this.lastDetection[exerciseType];
    if (!lastTime) return false;
    return now - lastTime < this.COOLDOWN_MS;
  }

  /**
   * データからピークを検出
   */
  private findPeaks(
    data: Array<{ time: number; value: number }>,
    threshold: number
  ): Array<{ time: number; value: number }> {
    const peaks: Array<{ time: number; value: number }> = [];

    for (let i = 1; i < data.length - 1; i++) {
      const prev = data[i - 1];
      const current = data[i];
      const next = data[i + 1];

      // 局所的な最大値かつしきい値を超えている
      if (current.value > prev.value &&
          current.value > next.value &&
          current.value >= threshold) {
        peaks.push(current);
      }
    }

    return peaks;
  }
}
```

### 使用例

```typescript
export const useMotionDetection = (
  exerciseType: ExerciseType,
  onDetection: (timestamp: number) => void
) => {
  const sensorManager = useRef(new MotionSensorManager()).current;

  useEffect(() => {
    sensorManager.start((detection) => {
      if (detection.type === exerciseType && detection.confidence > 0.8) {
        onDetection(detection.timestamp);
      }
    });

    return () => {
      sensorManager.stop();
    };
  }, [exerciseType, onDetection]);
};
```

---

## スコアリングシステム

### スコア計算定義

```typescript
/**
 * 判定ごとの基礎点
 */
export const BASE_SCORES = {
  [JudgmentType.PERFECT]: 1000,
  [JudgmentType.GREAT]: 500,
  [JudgmentType.GOOD]: 100,
  [JudgmentType.MISS]: 0,
} as const;

/**
 * コンボボーナス倍率
 * 10コンボごとに0.1倍増加、最大2.0倍
 */
const COMBO_BONUS_RATE = 0.1;
const COMBO_BONUS_STEP = 10;
const MAX_COMBO_BONUS = 2.0;

/**
 * 難易度倍率
 */
export enum Difficulty {
  NORMAL = 'NORMAL',
  HARD = 'HARD',
}

export const DIFFICULTY_MULTIPLIERS = {
  [Difficulty.NORMAL]: 1.0,
  [Difficulty.HARD]: 1.5,
} as const;

/**
 * スコア統計
 */
export interface ScoreStats {
  score: number;
  combo: number;
  maxCombo: number;
  judgmentCounts: Record<JudgmentType, number>;
  totalNotes: number;
}

/**
 * リザルト
 */
export interface GameResult extends ScoreStats {
  experiencePoints: number;
  rank: 'S' | 'A' | 'B' | 'C' | 'D';
  accuracy: number;        // 精度（0.0～1.0）
}
```

### スコア計算ロジック

```typescript
export class ScoreSystem {
  private score = 0;
  private combo = 0;
  private maxCombo = 0;
  private judgmentCounts: Record<JudgmentType, number> = {
    [JudgmentType.PERFECT]: 0,
    [JudgmentType.GREAT]: 0,
    [JudgmentType.GOOD]: 0,
    [JudgmentType.MISS]: 0,
  };

  /**
   * 判定結果を記録してスコアを加算
   */
  public addJudgment(judgment: JudgmentType): number {
    'worklet';

    // 判定カウントを増加
    this.judgmentCounts[judgment]++;

    // Missの場合はコンボをリセット
    if (judgment === JudgmentType.MISS) {
      this.combo = 0;
      return 0;
    }

    // コンボを増加
    this.combo++;
    this.maxCombo = Math.max(this.maxCombo, this.combo);

    // スコア計算
    const baseScore = BASE_SCORES[judgment];
    const comboBonus = this.calculateComboBonus();
    const finalScore = Math.floor(baseScore * comboBonus);

    this.score += finalScore;

    return finalScore;
  }

  /**
   * コンボボーナス倍率を計算
   */
  private calculateComboBonus(): number {
    'worklet';

    const bonusMultiplier = 1.0 + (Math.floor(this.combo / COMBO_BONUS_STEP) * COMBO_BONUS_RATE);
    return Math.min(bonusMultiplier, MAX_COMBO_BONUS);
  }

  /**
   * 現在のスコア統計を取得
   */
  public getStats(): ScoreStats {
    'worklet';

    const totalNotes = Object.values(this.judgmentCounts).reduce((a, b) => a + b, 0);

    return {
      score: this.score,
      combo: this.combo,
      maxCombo: this.maxCombo,
      judgmentCounts: { ...this.judgmentCounts },
      totalNotes,
    };
  }

  /**
   * ゲーム終了時のリザルトを計算
   */
  public calculateResult(difficulty: Difficulty, totalNotes: number): GameResult {
    'worklet';

    const stats = this.getStats();

    // 精度計算（Perfect=1.0, Great=0.5, Good=0.1, Miss=0.0）
    const perfectScore = this.judgmentCounts[JudgmentType.PERFECT] * 1.0;
    const greatScore = this.judgmentCounts[JudgmentType.GREAT] * 0.5;
    const goodScore = this.judgmentCounts[JudgmentType.GOOD] * 0.1;
    const accuracy = totalNotes > 0
      ? (perfectScore + greatScore + goodScore) / totalNotes
      : 0;

    // 経験値計算
    const difficultyMultiplier = DIFFICULTY_MULTIPLIERS[difficulty];
    const experiencePoints = Math.floor(this.score * difficultyMultiplier);

    // ランク判定
    const rank = this.calculateRank(accuracy);

    return {
      ...stats,
      experiencePoints,
      rank,
      accuracy,
    };
  }

  /**
   * ランク判定
   */
  private calculateRank(accuracy: number): 'S' | 'A' | 'B' | 'C' | 'D' {
    'worklet';

    if (accuracy >= 0.95) return 'S';
    if (accuracy >= 0.85) return 'A';
    if (accuracy >= 0.70) return 'B';
    if (accuracy >= 0.50) return 'C';
    return 'D';
  }

  /**
   * スコアシステムをリセット
   */
  public reset(): void {
    'worklet';

    this.score = 0;
    this.combo = 0;
    this.maxCombo = 0;
    this.judgmentCounts = {
      [JudgmentType.PERFECT]: 0,
      [JudgmentType.GREAT]: 0,
      [JudgmentType.GOOD]: 0,
      [JudgmentType.MISS]: 0,
    };
  }
}
```

### 経験値システム

```typescript
/**
 * 経験値とレベルの管理
 */
export class ExperienceSystem {
  /**
   * レベルアップに必要な経験値を計算
   * レベルnに到達するには: 100 * n^1.5
   */
  public static getRequiredExp(level: number): number {
    return Math.floor(100 * Math.pow(level, 1.5));
  }

  /**
   * 経験値から現在のレベルを計算
   */
  public static calculateLevel(totalExp: number): number {
    let level = 1;
    let expForNextLevel = this.getRequiredExp(level + 1);

    while (totalExp >= expForNextLevel) {
      level++;
      expForNextLevel = this.getRequiredExp(level + 1);
    }

    return level;
  }

  /**
   * 次のレベルまでの進捗を計算（0.0～1.0）
   */
  public static calculateProgress(totalExp: number): number {
    const currentLevel = this.calculateLevel(totalExp);
    const expForCurrentLevel = this.getRequiredExp(currentLevel);
    const expForNextLevel = this.getRequiredExp(currentLevel + 1);

    const expInCurrentLevel = totalExp - expForCurrentLevel;
    const expNeededForNextLevel = expForNextLevel - expForCurrentLevel;

    return expInCurrentLevel / expNeededForNextLevel;
  }
}
```

---

## ビートマップJSONスキーマ

### スキーマ定義

```typescript
/**
 * ノーツデータ
 */
export interface Note {
  id: string;                   // ユニークID
  targetTime: number;           // 判定タイミング（ミリ秒、曲開始からの経過時間）
  track: number;                // トラック番号（0-3）
  exerciseType: ExerciseType;   // 運動種目
  judged?: boolean;             // 判定済みフラグ
  judgmentResult?: JudgmentType;// 判定結果
}

/**
 * ビートマップメタデータ
 */
export interface BeatmapMetadata {
  title: string;                // 曲名
  artist: string;               // アーティスト名
  charter: string;              // 譜面制作者
  createdAt: string;            // 作成日時（ISO 8601）
  updatedAt: string;            // 更新日時（ISO 8601）
  version: string;              // ビートマップバージョン
}

/**
 * ビートマップデータ
 */
export interface Beatmap {
  // メタ情報
  songId: string;               // 曲ID（一意識別子）
  metadata: BeatmapMetadata;

  // 音楽情報
  bpm: number;                  // BPM（Beats Per Minute）
  offset: number;               // オフセット（ミリ秒、音楽開始から最初のビートまで）
  duration: number;             // 曲の長さ（ミリ秒）
  audioFile: string;            // 音楽ファイルパス

  // 譜面情報
  difficulty: Difficulty;       // 難易度
  level: number;                // レベル（1-10）
  notes: Note[];                // ノーツ配列

  // 統計情報（計算値）
  totalNotes: number;           // 総ノーツ数
  maxScore: number;             // 理論上の最高スコア
}

/**
 * ビートマップバンドル（複数難易度）
 */
export interface BeatmapBundle {
  songId: string;
  metadata: BeatmapMetadata;
  audioFile: string;
  difficulties: {
    [key in Difficulty]?: Beatmap;
  };
}
```

### JSON形式サンプル

```json
{
  "songId": "muscle-anthem-001",
  "metadata": {
    "title": "Muscle Anthem",
    "artist": "Fitness Beats",
    "charter": "MuscleMapper",
    "createdAt": "2026-02-01T10:00:00Z",
    "updatedAt": "2026-02-05T15:30:00Z",
    "version": "1.2"
  },
  "bpm": 140,
  "offset": 500,
  "duration": 180000,
  "audioFile": "songs/muscle-anthem-001.mp3",
  "difficulty": "NORMAL",
  "level": 5,
  "notes": [
    {
      "id": "note-001",
      "targetTime": 2000,
      "track": 0,
      "exerciseType": "SQUAT"
    },
    {
      "id": "note-002",
      "targetTime": 2428,
      "track": 1,
      "exerciseType": "SQUAT"
    },
    {
      "id": "note-003",
      "targetTime": 2857,
      "track": 2,
      "exerciseType": "SIT_UP"
    },
    {
      "id": "note-004",
      "targetTime": 3285,
      "track": 3,
      "exerciseType": "SQUAT"
    },
    {
      "id": "note-005",
      "targetTime": 4000,
      "track": 0,
      "exerciseType": "BURPEE"
    }
  ],
  "totalNotes": 5,
  "maxScore": 10000
}
```

### 複数難易度のバンドル形式

```json
{
  "songId": "muscle-anthem-001",
  "metadata": {
    "title": "Muscle Anthem",
    "artist": "Fitness Beats",
    "charter": "MuscleMapper",
    "createdAt": "2026-02-01T10:00:00Z",
    "updatedAt": "2026-02-05T15:30:00Z",
    "version": "1.2"
  },
  "audioFile": "songs/muscle-anthem-001.mp3",
  "difficulties": {
    "NORMAL": {
      "songId": "muscle-anthem-001",
      "metadata": { },
      "bpm": 140,
      "offset": 500,
      "duration": 180000,
      "audioFile": "songs/muscle-anthem-001.mp3",
      "difficulty": "NORMAL",
      "level": 5,
      "notes": [ ],
      "totalNotes": 50,
      "maxScore": 10000
    },
    "HARD": {
      "songId": "muscle-anthem-001",
      "metadata": { },
      "bpm": 140,
      "offset": 500,
      "duration": 180000,
      "audioFile": "songs/muscle-anthem-001.mp3",
      "difficulty": "HARD",
      "level": 8,
      "notes": [ ],
      "totalNotes": 120,
      "maxScore": 25000
    }
  }
}
```

### ビートマップローダー

```typescript
/**
 * ビートマップをロードするユーティリティ
 */
export class BeatmapLoader {
  /**
   * JSONからビートマップを読み込み
   */
  public static async loadBeatmap(songId: string, difficulty: Difficulty): Promise<Beatmap> {
    try {
      const response = await fetch(`/beatmaps/${songId}.json`);
      const bundle: BeatmapBundle = await response.json();

      const beatmap = bundle.difficulties[difficulty];
      if (!beatmap) {
        throw new Error(`Difficulty ${difficulty} not found for song ${songId}`);
      }

      // バリデーション
      this.validateBeatmap(beatmap);

      return beatmap;
    } catch (error) {
      console.error('Failed to load beatmap:', error);
      throw error;
    }
  }

  /**
   * ビートマップの整合性をチェック
   */
  private static validateBeatmap(beatmap: Beatmap): void {
    // 必須フィールドの存在チェック
    if (!beatmap.songId || !beatmap.bpm || !beatmap.notes) {
      throw new Error('Invalid beatmap: missing required fields');
    }

    // BPM範囲チェック
    if (beatmap.bpm < 60 || beatmap.bpm > 240) {
      throw new Error(`Invalid BPM: ${beatmap.bpm}`);
    }

    // ノーツの時間順ソート確認
    for (let i = 1; i < beatmap.notes.length; i++) {
      if (beatmap.notes[i].targetTime < beatmap.notes[i - 1].targetTime) {
        throw new Error('Notes are not sorted by time');
      }
    }

    // ノーツ数の整合性
    if (beatmap.notes.length !== beatmap.totalNotes) {
      console.warn('Note count mismatch, recalculating...');
      beatmap.totalNotes = beatmap.notes.length;
    }
  }

  /**
   * 利用可能な曲リストを取得
   */
  public static async getSongList(): Promise<Array<{ songId: string; metadata: BeatmapMetadata }>> {
    const response = await fetch('/beatmaps/index.json');
    const index = await response.json();
    return index.songs;
  }
}
```

---

## 音楽同期メカニズム

### 音楽再生管理

```typescript
import { Audio, AVPlaybackStatus, AVPlaybackStatusSuccess } from 'expo-av';
import { SharedValue } from 'react-native-reanimated';

/**
 * 音楽プレイヤー
 */
export class MusicPlayer {
  private sound: Audio.Sound | null = null;
  private isPlaying = false;
  private playbackRate = 1.0;

  // 音楽位置をSharedValueで管理（ワークレットからアクセス可能）
  public positionMs: SharedValue<number>;

  constructor(positionMs: SharedValue<number>) {
    this.positionMs = positionMs;
  }

  /**
   * 音楽をロード
   */
  public async load(audioFile: string): Promise<void> {
    // 既存のサウンドをアンロード
    if (this.sound) {
      await this.sound.unloadAsync();
    }

    // オーディオモードを設定
    await Audio.setAudioModeAsync({
      playsInSilentModeIOS: true,
      staysActiveInBackground: false,
      shouldDuckAndroid: true,
    });

    // サウンドをロード
    const { sound } = await Audio.Sound.createAsync(
      { uri: audioFile },
      {
        shouldPlay: false,
        progressUpdateIntervalMillis: 16, // 約60FPSで更新
      },
      this.onPlaybackStatusUpdate.bind(this)
    );

    this.sound = sound;
  }

  /**
   * 再生開始
   */
  public async play(): Promise<void> {
    if (!this.sound) {
      throw new Error('Sound not loaded');
    }

    await this.sound.playAsync();
    this.isPlaying = true;
  }

  /**
   * 一時停止
   */
  public async pause(): Promise<void> {
    if (!this.sound) return;

    await this.sound.pauseAsync();
    this.isPlaying = false;
  }

  /**
   * 停止
   */
  public async stop(): Promise<void> {
    if (!this.sound) return;

    await this.sound.stopAsync();
    this.isPlaying = false;
    this.positionMs.value = 0;
  }

  /**
   * シーク
   */
  public async seek(positionMs: number): Promise<void> {
    if (!this.sound) return;

    await this.sound.setPositionAsync(positionMs);
  }

  /**
   * 再生速度を設定
   */
  public async setPlaybackRate(rate: number): Promise<void> {
    if (!this.sound) return;

    await this.sound.setRateAsync(rate, true);
    this.playbackRate = rate;
  }

  /**
   * 再生状態の更新コールバック
   */
  private onPlaybackStatusUpdate(status: AVPlaybackStatus): void {
    if (status.isLoaded) {
      const successStatus = status as AVPlaybackStatusSuccess;

      // SharedValueを更新（ワークレットからアクセス可能）
      this.positionMs.value = successStatus.positionMillis;

      // 再生終了時
      if (successStatus.didJustFinish && !successStatus.isLooping) {
        this.isPlaying = false;
      }
    }
  }

  /**
   * リソースをクリーンアップ
   */
  public async unload(): Promise<void> {
    if (this.sound) {
      await this.sound.unloadAsync();
      this.sound = null;
    }
  }

  /**
   * 現在の再生位置を取得
   */
  public async getCurrentPosition(): Promise<number> {
    if (!this.sound) return 0;

    const status = await this.sound.getStatusAsync();
    if (status.isLoaded) {
      return status.positionMillis;
    }
    return 0;
  }
}
```

### 遅延補正システム

```typescript
/**
 * レイテンシー測定・補正
 */
export class LatencyCalibration {
  private audioLatencyMs = 0;    // オーディオ遅延
  private visualLatencyMs = 0;   // 視覚遅延
  private inputLatencyMs = 0;    // 入力遅延

  /**
   * 総合オフセットを取得
   */
  public getTotalOffset(): number {
    return this.audioLatencyMs + this.visualLatencyMs + this.inputLatencyMs;
  }

  /**
   * オーディオレイテンシーを設定
   */
  public setAudioLatency(latencyMs: number): void {
    this.audioLatencyMs = latencyMs;
  }

  /**
   * 視覚レイテンシーを設定
   */
  public setVisualLatency(latencyMs: number): void {
    this.visualLatencyMs = latencyMs;
  }

  /**
   * 入力レイテンシーを設定
   */
  public setInputLatency(latencyMs: number): void {
    this.inputLatencyMs = latencyMs;
  }

  /**
   * 自動キャリブレーション
   * ユーザーにタップを促し、平均遅延を計算
   */
  public async performAutoCalibration(
    onPrompt: (beatTime: number) => void,
    onTap: () => Promise<number>
  ): Promise<number> {
    const BPM = 120;
    const BEAT_INTERVAL_MS = 60000 / BPM;
    const NUM_SAMPLES = 10;

    const deviations: number[] = [];

    for (let i = 0; i < NUM_SAMPLES; i++) {
      const beatTime = Date.now();
      onPrompt(beatTime);

      // ユーザーのタップを待つ
      const tapTime = await onTap();
      const deviation = tapTime - beatTime;

      deviations.push(deviation);

      // 次のビートまで待機
      await new Promise(resolve => setTimeout(resolve, BEAT_INTERVAL_MS));
    }

    // 平均遅延を計算
    const averageDeviation = deviations.reduce((a, b) => a + b, 0) / deviations.length;

    // 入力レイテンシーとして設定
    this.setInputLatency(averageDeviation);

    return averageDeviation;
  }
}
```

### 同期調整された時間取得

```typescript
/**
 * 同期時間マネージャー
 */
export class SyncTimeManager {
  private musicPlayer: MusicPlayer;
  private latencyCalibration: LatencyCalibration;
  private beatmapOffset: number;

  constructor(
    musicPlayer: MusicPlayer,
    latencyCalibration: LatencyCalibration,
    beatmapOffset: number
  ) {
    this.musicPlayer = musicPlayer;
    this.latencyCalibration = latencyCalibration;
    this.beatmapOffset = beatmapOffset;
  }

  /**
   * 補正済みの現在時刻を取得（ワークレット内で使用）
   */
  public getCalibratedTime(): number {
    'worklet';

    const rawPosition = this.musicPlayer.positionMs.value;
    const totalOffset = this.latencyCalibration.getTotalOffset();
    const beatmapOffset = this.beatmapOffset;

    // 補正を適用
    return rawPosition - totalOffset + beatmapOffset;
  }

  /**
   * ビート番号を計算
   */
  public getBeatNumber(bpm: number): number {
    'worklet';

    const timeMs = this.getCalibratedTime();
    const beatDurationMs = 60000 / bpm;
    return Math.floor(timeMs / beatDurationMs);
  }
}
```

---

## ステートマシン

### ゲームステート定義

```typescript
/**
 * ゲーム全体のステート
 */
export enum GameState {
  READY = 'READY',               // 準備完了（スタート前）
  COUNTDOWN = 'COUNTDOWN',       // カウントダウン中（3, 2, 1, GO!）
  PLAYING = 'PLAYING',           // プレイ中
  PAUSED = 'PAUSED',             // 一時停止
  FINISHED = 'FINISHED',         // 終了
}

/**
 * ステート遷移イベント
 */
export enum GameEvent {
  START = 'START',               // ゲーム開始
  COUNTDOWN_COMPLETE = 'COUNTDOWN_COMPLETE', // カウントダウン完了
  PAUSE = 'PAUSE',               // 一時停止
  RESUME = 'RESUME',             // 再開
  FINISH = 'FINISH',             // 終了
  RESTART = 'RESTART',           // 再スタート
}

/**
 * ステート遷移定義
 */
const STATE_TRANSITIONS: Record<GameState, Partial<Record<GameEvent, GameState>>> = {
  [GameState.READY]: {
    [GameEvent.START]: GameState.COUNTDOWN,
  },
  [GameState.COUNTDOWN]: {
    [GameEvent.COUNTDOWN_COMPLETE]: GameState.PLAYING,
  },
  [GameState.PLAYING]: {
    [GameEvent.PAUSE]: GameState.PAUSED,
    [GameEvent.FINISH]: GameState.FINISHED,
  },
  [GameState.PAUSED]: {
    [GameEvent.RESUME]: GameState.PLAYING,
    [GameEvent.FINISH]: GameState.FINISHED,
  },
  [GameState.FINISHED]: {
    [GameEvent.RESTART]: GameState.READY,
  },
};
```

### ステートマシン実装

```typescript
/**
 * ゲームステートマシン
 */
export class GameStateMachine {
  private currentState: GameState = GameState.READY;
  private listeners: Array<(state: GameState) => void> = [];

  /**
   * 現在のステートを取得
   */
  public getState(): GameState {
    return this.currentState;
  }

  /**
   * イベントを送信してステート遷移
   */
  public sendEvent(event: GameEvent): boolean {
    const nextState = STATE_TRANSITIONS[this.currentState]?.[event];

    if (!nextState) {
      console.warn(`Invalid transition: ${this.currentState} + ${event}`);
      return false;
    }

    console.log(`State transition: ${this.currentState} -> ${nextState}`);

    // ステート退出処理
    this.onExitState(this.currentState);

    // ステート更新
    const previousState = this.currentState;
    this.currentState = nextState;

    // ステート進入処理
    this.onEnterState(nextState, previousState);

    // リスナーに通知
    this.notifyListeners(nextState);

    return true;
  }

  /**
   * ステート変更リスナーを追加
   */
  public addListener(listener: (state: GameState) => void): () => void {
    this.listeners.push(listener);

    // アンサブスクライブ関数を返す
    return () => {
      this.listeners = this.listeners.filter(l => l !== listener);
    };
  }

  /**
   * ステート進入時の処理
   */
  private onEnterState(state: GameState, previousState: GameState): void {
    console.log(`Entering state: ${state}`);

    // ステートごとの初期化処理
    switch (state) {
      case GameState.COUNTDOWN:
        // カウントダウン開始
        break;
      case GameState.PLAYING:
        // ゲーム開始
        break;
      case GameState.FINISHED:
        // リザルト計算
        break;
    }
  }

  /**
   * ステート退出時の処理
   */
  private onExitState(state: GameState): void {
    console.log(`Exiting state: ${state}`);

    // ステートごとのクリーンアップ処理
    switch (state) {
      case GameState.PLAYING:
        // 一時停止処理
        break;
    }
  }

  /**
   * リスナーに通知
   */
  private notifyListeners(state: GameState): void {
    this.listeners.forEach(listener => listener(state));
  }

  /**
   * ステートマシンをリセット
   */
  public reset(): void {
    this.currentState = GameState.READY;
    this.notifyListeners(this.currentState);
  }
}
```

### カウントダウン実装

```typescript
/**
 * カウントダウンマネージャー
 */
export class CountdownManager {
  private countdownValue = 3;
  private timer: NodeJS.Timeout | null = null;

  /**
   * カウントダウンを開始
   * @param onTick カウント更新時のコールバック
   * @param onComplete 完了時のコールバック
   */
  public start(
    onTick: (count: number) => void,
    onComplete: () => void
  ): void {
    this.countdownValue = 3;

    onTick(this.countdownValue);

    this.timer = setInterval(() => {
      this.countdownValue--;

      if (this.countdownValue > 0) {
        onTick(this.countdownValue);
      } else {
        this.stop();
        onComplete();
      }
    }, 1000);
  }

  /**
   * カウントダウンを停止
   */
  public stop(): void {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }

  /**
   * 現在のカウント値を取得
   */
  public getCurrentCount(): number {
    return this.countdownValue;
  }
}
```

### ゲームコントローラー統合

```typescript
/**
 * ゲーム全体を制御するコントローラー
 */
export class GameController {
  private stateMachine: GameStateMachine;
  private musicPlayer: MusicPlayer;
  private gameEngine: GameEngine;
  private countdownManager: CountdownManager;

  constructor(
    musicPlayer: MusicPlayer,
    gameEngine: GameEngine
  ) {
    this.stateMachine = new GameStateMachine();
    this.musicPlayer = musicPlayer;
    this.gameEngine = gameEngine;
    this.countdownManager = new CountdownManager();

    // ステート変更リスナー
    this.stateMachine.addListener(this.onStateChanged.bind(this));
  }

  /**
   * ゲームを開始
   */
  public async start(): Promise<void> {
    // READY -> COUNTDOWN
    this.stateMachine.sendEvent(GameEvent.START);
  }

  /**
   * ゲームを一時停止
   */
  public async pause(): Promise<void> {
    this.stateMachine.sendEvent(GameEvent.PAUSE);
  }

  /**
   * ゲームを再開
   */
  public async resume(): Promise<void> {
    this.stateMachine.sendEvent(GameEvent.RESUME);
  }

  /**
   * ゲームを終了
   */
  public async finish(): Promise<void> {
    this.stateMachine.sendEvent(GameEvent.FINISH);
  }

  /**
   * ステート変更時の処理
   */
  private async onStateChanged(state: GameState): Promise<void> {
    switch (state) {
      case GameState.COUNTDOWN:
        await this.handleCountdown();
        break;

      case GameState.PLAYING:
        await this.handlePlaying();
        break;

      case GameState.PAUSED:
        await this.handlePaused();
        break;

      case GameState.FINISHED:
        await this.handleFinished();
        break;
    }
  }

  /**
   * カウントダウン処理
   */
  private async handleCountdown(): Promise<void> {
    this.countdownManager.start(
      (count) => {
        console.log(`Countdown: ${count}`);
        // UIにカウント表示
      },
      () => {
        // カウントダウン完了 -> PLAYING
        this.stateMachine.sendEvent(GameEvent.COUNTDOWN_COMPLETE);
      }
    );
  }

  /**
   * プレイ中処理
   */
  private async handlePlaying(): Promise<void> {
    // 音楽再生開始
    await this.musicPlayer.play();

    // ゲームエンジン開始
    this.gameEngine.start();
  }

  /**
   * 一時停止処理
   */
  private async handlePaused(): Promise<void> {
    // 音楽一時停止
    await this.musicPlayer.pause();

    // ゲームエンジン一時停止
    this.gameEngine.pause();
  }

  /**
   * 終了処理
   */
  private async handleFinished(): Promise<void> {
    // 音楽停止
    await this.musicPlayer.stop();

    // ゲームエンジン停止
    this.gameEngine.stop();

    // リザルト計算
    const result = this.gameEngine.getResult();
    console.log('Game finished:', result);
  }
}
```

---

## パフォーマンス最適化

### 60FPS維持のための戦略

#### 1. ワークレットの活用

```typescript
/**
 * 重い計算をワークレットで実行
 */
import { runOnUI } from 'react-native-reanimated';

// UIスレッドをブロックせずにノーツ判定
const performJudgment = (note: Note, inputTime: number) => {
  'worklet';

  // 判定ロジック（ワークレットスレッドで実行）
  const judgment = judgmentSystem.judge(note, inputTime);

  // 結果をUIスレッドに送信
  runOnJS(updateUI)(judgment);
};

// ゲームループから呼び出し
runOnUI(performJudgment)(note, inputTime);
```

#### 2. メモ化とキャッシング

```typescript
/**
 * ノーツ描画の最適化
 */
export const NoteComponent = React.memo<{ note: Note; currentTime: number }>(
  ({ note, currentTime }) => {
    // アニメーション値をキャッシュ
    const translateY = useDerivedValue(() => {
      'worklet';
      const timeUntilHit = note.targetTime - currentTime;
      const FALL_SPEED = 500; // px/s
      return timeUntilHit * FALL_SPEED / 1000;
    }, [note.targetTime, currentTime]);

    const animatedStyle = useAnimatedStyle(() => ({
      transform: [{ translateY: translateY.value }],
    }));

    return (
      <Animated.View style={[styles.note, animatedStyle]}>
        {/* ノーツ描画 */}
      </Animated.View>
    );
  },
  (prev, next) => {
    // targetTimeが同じなら再レンダリングしない
    return prev.note.id === next.note.id;
  }
);
```

#### 3. オブジェクトプーリング

```typescript
/**
 * ノーツオブジェクトのプール
 */
export class NotePool {
  private pool: Note[] = [];
  private activeNotes: Set<Note> = new Set();

  /**
   * ノーツを取得（再利用）
   */
  public acquire(targetTime: number, track: number, exerciseType: ExerciseType): Note {
    let note: Note;

    if (this.pool.length > 0) {
      // プールから再利用
      note = this.pool.pop()!;
      note.targetTime = targetTime;
      note.track = track;
      note.exerciseType = exerciseType;
      note.judged = false;
      note.judgmentResult = undefined;
    } else {
      // 新規作成
      note = {
        id: `note-${Date.now()}-${Math.random()}`,
        targetTime,
        track,
        exerciseType,
      };
    }

    this.activeNotes.add(note);
    return note;
  }

  /**
   * ノーツを返却（プールに戻す）
   */
  public release(note: Note): void {
    if (this.activeNotes.has(note)) {
      this.activeNotes.delete(note);
      this.pool.push(note);
    }
  }

  /**
   * プールをクリア
   */
  public clear(): void {
    this.pool = [];
    this.activeNotes.clear();
  }
}
```

#### 4. レンダリング最適化

```typescript
/**
 * 可視範囲のノーツのみレンダリング
 */
export const NotesRenderer: React.FC<{
  notes: Note[];
  currentTime: number;
}> = ({ notes, currentTime }) => {
  // 可視範囲のノーツをフィルタリング
  const visibleNotes = useMemo(() => {
    const VISIBLE_WINDOW_MS = 3000; // 3秒先まで表示
    const windowStart = currentTime;
    const windowEnd = currentTime + VISIBLE_WINDOW_MS;

    return notes.filter(
      note => !note.judged &&
              note.targetTime >= windowStart &&
              note.targetTime <= windowEnd
    );
  }, [notes, currentTime]);

  return (
    <View style={styles.notesContainer}>
      {visibleNotes.map(note => (
        <NoteComponent key={note.id} note={note} currentTime={currentTime} />
      ))}
    </View>
  );
};
```

#### 5. センサーデータのスロットリング

```typescript
/**
 * センサーデータの間引き
 */
export class ThrottledSensorManager extends MotionSensorManager {
  private lastProcessedTime = 0;
  private readonly THROTTLE_MS = 16; // 約60FPS

  protected addSample(sample: SensorSample): void {
    const now = sample.timestamp;

    // スロットリング
    if (now - this.lastProcessedTime < this.THROTTLE_MS) {
      return;
    }

    this.lastProcessedTime = now;
    super.addSample(sample);
  }
}
```

### パフォーマンス計測

```typescript
/**
 * フレームレート計測
 */
export class PerformanceMonitor {
  private frameTimes: number[] = [];
  private lastFrameTime = 0;

  /**
   * フレーム記録
   */
  public recordFrame(): void {
    'worklet';

    const now = Date.now();

    if (this.lastFrameTime > 0) {
      const frameTime = now - this.lastFrameTime;
      this.frameTimes.push(frameTime);

      // 直近100フレームのみ保持
      if (this.frameTimes.length > 100) {
        this.frameTimes.shift();
      }
    }

    this.lastFrameTime = now;
  }

  /**
   * 平均FPSを取得
   */
  public getAverageFPS(): number {
    'worklet';

    if (this.frameTimes.length === 0) return 0;

    const avgFrameTime = this.frameTimes.reduce((a, b) => a + b, 0) / this.frameTimes.length;
    return 1000 / avgFrameTime;
  }

  /**
   * フレームドロップを検出
   */
  public getDroppedFrames(): number {
    'worklet';

    const TARGET_FRAME_TIME = 1000 / 60; // 16.67ms
    const DROPPED_THRESHOLD = TARGET_FRAME_TIME * 2; // 2フレーム分

    return this.frameTimes.filter(time => time > DROPPED_THRESHOLD).length;
  }
}
```

---

## まとめ

本ドキュメントでは、Muscle Beatのゲームエンジンの詳細設計を定義しました。

### 主要コンポーネント

1. **ゲームループ**: `useFrameCallback`による60FPS安定動作
2. **判定システム**: ±50ms/±120ms/±265msの3段階判定
3. **センサー連携**: 60Hzサンプリング、種目別しきい値検知
4. **スコアリング**: コンボボーナス、経験値変換
5. **ビートマップ**: JSON形式、複数難易度対応
6. **音楽同期**: expo-av + 遅延補正
7. **ステートマシン**: 明確な状態遷移管理
8. **パフォーマンス**: ワークレット、メモ化、オブジェクトプーリング

### 実装の流れ

1. 基本型定義とインターフェースの実装
2. ゲームループとフレーム処理の構築
3. 判定システムとモーションセンサーの統合
4. スコアリングとビートマップローダーの実装
5. 音楽プレイヤーと同期システムの構築
6. ステートマシンとゲームコントローラーの実装
7. パフォーマンス最適化とテスト

### 次のステップ

- 各コンポーネントの単体テスト作成
- 統合テストとパフォーマンステスト
- ビートマップエディターの開発
- 難易度調整とバランス調整
- ユーザーフィードバックに基づく改善

このドキュメントに基づいて実装を進めることで、安定した60FPS動作と正確な判定を実現する高品質なゲームエンジンを構築できます。
