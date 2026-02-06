# Muscle Beat アーキテクチャ設計書

## 1. プロジェクト概要

Muscle Beatは、モーションセンサーを使った音楽ゲームとキャラクター育成を組み合わせたReact Nativeアプリケーションです。プレイヤーは楽曲をプレイすることで経験値を獲得し、キャラクターを7段階進化させます。

## 2. ディレクトリ構成

```
src/
├── screens/              # 画面コンポーネント
│   ├── HomeScreen.tsx                # ホーム画面（キャラクター表示）
│   ├── SongSelectScreen.tsx          # 曲選択画面
│   ├── GamePrepareScreen.tsx         # ゲーム準備画面
│   ├── GamePlayScreen.tsx            # ゲームプレイ画面
│   ├── ResultScreen.tsx              # リザルト画面
│   └── TrainingRecordScreen.tsx      # トレーニング記録画面
│
├── components/           # 再利用可能なコンポーネント
│   ├── character/
│   │   ├── CharacterDisplay.tsx      # キャラクター表示
│   │   ├── EvolutionAnimation.tsx    # 進化アニメーション
│   │   └── ExpBar.tsx                # 経験値バー
│   ├── game/
│   │   ├── NoteTrack.tsx             # ノーツトラック表示
│   │   ├── Note.tsx                  # 個別ノーツ
│   │   ├── JudgementLine.tsx         # 判定ライン
│   │   ├── ComboDisplay.tsx          # コンボ表示
│   │   └── ScoreDisplay.tsx          # スコア表示
│   ├── ui/
│   │   ├── Button.tsx                # 共通ボタン
│   │   ├── Card.tsx                  # カード型コンテナ
│   │   └── Header.tsx                # ヘッダー
│   └── record/
│       ├── CalendarView.tsx          # カレンダー表示
│       └── RecordListItem.tsx        # 記録リストアイテム
│
├── store/                # Zustand状態管理
│   ├── characterStore.ts             # キャラクター状態
│   ├── gameStore.ts                  # ゲームプレイ状態
│   └── recordStore.ts                # トレーニング記録状態
│
├── database/             # データベース層
│   ├── index.ts                      # DB初期化とエクスポート
│   ├── schema.ts                     # テーブルスキーマ定義
│   ├── migrations.ts                 # マイグレーション管理
│   └── repositories/
│       ├── trainingRecordRepository.ts  # トレーニング記録CRUD
│       └── characterStateRepository.ts  # キャラクター状態CRUD
│
├── services/             # ビジネスロジック
│   ├── gameEngine.ts                 # ゲームロジック（判定、スコア計算）
│   ├── motionDetector.ts             # モーション検出サービス
│   ├── audioPlayer.ts                # 音楽再生サービス
│   ├── experienceCalculator.ts       # 経験値計算ロジック
│   ├── evolutionManager.ts           # 進化管理ロジック
│   └── degradationService.ts         # 退化チェックロジック
│
├── data/                 # 静的データ
│   ├── songs.ts                      # 楽曲データ定義
│   ├── noteCharts.ts                 # ノーツ譜面データ
│   └── evolutionStages.ts            # 進化段階定義
│
├── types/                # TypeScript型定義
│   ├── game.ts                       # ゲーム関連型
│   ├── character.ts                  # キャラクター関連型
│   ├── database.ts                   # データベース関連型
│   └── navigation.ts                 # ナビゲーション型
│
├── hooks/                # カスタムフック
│   ├── useMotionSensor.ts            # モーションセンサーフック
│   ├── useGameTimer.ts               # ゲームタイマーフック
│   ├── useAudio.ts                   # 音楽再生フック
│   └── useCharacterEvolution.ts      # キャラクター進化フック
│
├── utils/                # ユーティリティ関数
│   ├── dateUtils.ts                  # 日付計算
│   ├── scoreUtils.ts                 # スコア計算補助
│   └── constants.ts                  # 定数定義
│
├── assets/               # アセット
│   ├── animations/                   # Lottieアニメーション
│   ├── images/                       # キャラクター画像
│   └── audio/                        # 楽曲ファイル
│
└── navigation/           # ナビゲーション設定
    └── RootNavigator.tsx             # ルートナビゲーター
```

## 3. 状態管理設計（Zustand）

### 3.1 characterStore.ts

キャラクターの状態を管理します。

```typescript
interface CharacterState {
  // 状態
  id: number;
  totalExp: number;
  evolutionStage: number; // 1-7
  lastPlayedAt: Date | null;

  // アクション
  addExp: (exp: number) => Promise<void>;
  checkAndApplyDegradation: () => Promise<void>;
  loadCharacter: () => Promise<void>;
  resetCharacter: () => Promise<void>;
}
```

**主な責務:**
- 経験値の追加と進化判定
- 退化チェックと適用
- DBからのキャラクター状態読み込み

### 3.2 gameStore.ts

ゲームプレイ中の状態を管理します。

```typescript
interface GameState {
  // ゲーム設定
  selectedSongId: string | null;
  difficulty: 'Normal' | 'Hard' | null;

  // プレイ状態
  isPlaying: boolean;
  isPaused: boolean;
  currentTime: number;

  // スコア・判定
  score: number;
  combo: number;
  maxCombo: number;
  perfectCount: number;
  greatCount: number;
  goodCount: number;
  missCount: number;

  // ノーツ管理
  notes: Note[];
  activeNoteIndex: number;

  // アクション
  startGame: (songId: string, difficulty: 'Normal' | 'Hard') => void;
  pauseGame: () => void;
  resumeGame: () => void;
  updateTime: (time: number) => void;
  judgeNote: (timing: number, noteType: string) => JudgementResult;
  finishGame: () => Promise<GameResult>;
  resetGame: () => void;
}
```

**主な責務:**
- ゲームフローの制御
- リアルタイムスコア計算
- ノーツ判定処理
- ゲーム終了時の結果保存

### 3.3 recordStore.ts

トレーニング記録の状態を管理します。

```typescript
interface RecordState {
  // 状態
  records: TrainingRecord[];
  isLoading: boolean;

  // フィルタ
  selectedMonth: Date;

  // アクション
  loadRecords: (startDate?: Date, endDate?: Date) => Promise<void>;
  addRecord: (record: Omit<TrainingRecord, 'id'>) => Promise<void>;
  getRecordsByDate: (date: Date) => TrainingRecord[];
  getBestScore: (songId: string, difficulty: string) => number | null;
  getTotalPlayCount: () => number;
}
```

**主な責務:**
- トレーニング記録の取得と表示
- 日付範囲でのフィルタリング
- 統計情報の提供

## 4. データベーススキーマ

### 4.1 DDL定義

```sql
-- トレーニング記録テーブル
CREATE TABLE IF NOT EXISTS training_records (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  song_id TEXT NOT NULL,
  difficulty TEXT NOT NULL CHECK(difficulty IN ('Normal', 'Hard')),
  score INTEGER NOT NULL CHECK(score >= 0),
  exp_earned INTEGER NOT NULL CHECK(exp_earned >= 0),
  perfect_count INTEGER NOT NULL DEFAULT 0,
  great_count INTEGER NOT NULL DEFAULT 0,
  good_count INTEGER NOT NULL DEFAULT 0,
  miss_count INTEGER NOT NULL DEFAULT 0,
  max_combo INTEGER NOT NULL DEFAULT 0,
  played_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- キャラクター状態テーブル（常に1レコード）
CREATE TABLE IF NOT EXISTS character_state (
  id INTEGER PRIMARY KEY CHECK(id = 1),
  total_exp INTEGER NOT NULL DEFAULT 0,
  evolution_stage INTEGER NOT NULL DEFAULT 1 CHECK(evolution_stage BETWEEN 1 AND 7),
  last_played_at DATETIME,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- インデックス
CREATE INDEX IF NOT EXISTS idx_training_records_played_at ON training_records(played_at DESC);
CREATE INDEX IF NOT EXISTS idx_training_records_song ON training_records(song_id, difficulty, score DESC);

-- 初期データ挿入
INSERT OR IGNORE INTO character_state (id, total_exp, evolution_stage)
VALUES (1, 0, 1);
```

### 4.2 型定義

```typescript
// database.ts
export interface TrainingRecord {
  id: number;
  song_id: string;
  difficulty: 'Normal' | 'Hard';
  score: number;
  exp_earned: number;
  perfect_count: number;
  great_count: number;
  good_count: number;
  miss_count: number;
  max_combo: number;
  played_at: string; // ISO 8601形式
  created_at: string;
}

export interface CharacterState {
  id: 1;
  total_exp: number;
  evolution_stage: number; // 1-7
  last_played_at: string | null;
  created_at: string;
  updated_at: string;
}
```

## 5. 画面遷移設計（React Navigation Stack）

### 5.1 ナビゲーション型定義

```typescript
// types/navigation.ts
export type RootStackParamList = {
  Home: undefined;
  SongSelect: undefined;
  GamePrepare: {
    songId: string;
    difficulty: 'Normal' | 'Hard';
  };
  GamePlay: {
    songId: string;
    difficulty: 'Normal' | 'Hard';
  };
  Result: {
    score: number;
    expEarned: number;
    perfectCount: number;
    greatCount: number;
    goodCount: number;
    missCount: number;
    maxCombo: number;
    didEvolve: boolean;
    newStage?: number;
  };
  TrainingRecord: undefined;
};
```

### 5.2 画面遷移フロー

```
Home
  ├─→ SongSelect
  │     └─→ GamePrepare
  │           └─→ GamePlay
  │                 └─→ Result
  │                       └─→ Home (リザルト後は必ずホームへ)
  └─→ TrainingRecord
        └─→ Home
```

**遷移ルール:**
- ホーム画面がルート（戻るボタンなし）
- ゲームプレイ中は戻る操作を無効化
- リザルト画面からは必ずホームに戻る（履歴クリア）

### 5.3 RootNavigator実装例

```typescript
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

const Stack = createStackNavigator<RootStackParamList>();

export const RootNavigator = () => {
  return (
    <NavigationContainer>
      <Stack.Navigator
        initialRouteName="Home"
        screenOptions={{
          headerShown: false,
          gestureEnabled: false, // スワイプバック無効
        }}
      >
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="SongSelect" component={SongSelectScreen} />
        <Stack.Screen name="GamePrepare" component={GamePrepareScreen} />
        <Stack.Screen
          name="GamePlay"
          component={GamePlayScreen}
          options={{ gestureEnabled: false }} // 絶対に戻れないようにする
        />
        <Stack.Screen
          name="Result"
          component={ResultScreen}
          options={{ gestureEnabled: false }}
        />
        <Stack.Screen name="TrainingRecord" component={TrainingRecordScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
};
```

## 6. コアサービス設計

### 6.1 experienceCalculator.ts

経験値計算ロジック。

```typescript
export interface ExpCalculationParams {
  score: number;
  difficulty: 'Normal' | 'Hard';
}

export const calculateExp = (params: ExpCalculationParams): number => {
  const { score, difficulty } = params;
  const difficultyMultiplier = difficulty === 'Hard' ? 1.5 : 1.0;
  return Math.floor(score * difficultyMultiplier);
};
```

### 6.2 evolutionManager.ts

進化段階管理ロジック。

```typescript
export interface EvolutionStage {
  stage: number;
  name: string;
  requiredExp: number;
  image: string;
  animation?: string;
}

export const EVOLUTION_STAGES: EvolutionStage[] = [
  { stage: 1, name: 'タマゴ', requiredExp: 0, image: 'egg.png' },
  { stage: 2, name: '赤ちゃん', requiredExp: 3000, image: 'baby.png' },
  { stage: 3, name: '子ども', requiredExp: 10000, image: 'child.png' },
  { stage: 4, name: '大人', requiredExp: 25000, image: 'adult.png' },
  { stage: 5, name: 'マッチョ', requiredExp: 50000, image: 'macho.png' },
  { stage: 6, name: 'ボディビルダー', requiredExp: 85000, image: 'bodybuilder.png' },
  { stage: 7, name: 'ゴリラ', requiredExp: 130000, image: 'gorilla.png' },
];

export const getStageByExp = (totalExp: number): number => {
  for (let i = EVOLUTION_STAGES.length - 1; i >= 0; i--) {
    if (totalExp >= EVOLUTION_STAGES[i].requiredExp) {
      return EVOLUTION_STAGES[i].stage;
    }
  }
  return 1;
};

export const getNextStageExp = (currentStage: number): number | null => {
  if (currentStage >= 7) return null;
  return EVOLUTION_STAGES[currentStage].requiredExp;
};

export const getExpProgress = (totalExp: number, currentStage: number): number => {
  if (currentStage >= 7) return 1.0;

  const currentStageExp = EVOLUTION_STAGES[currentStage - 1].requiredExp;
  const nextStageExp = EVOLUTION_STAGES[currentStage].requiredExp;
  const expInCurrentStage = totalExp - currentStageExp;
  const expNeeded = nextStageExp - currentStageExp;

  return Math.min(expInCurrentStage / expNeeded, 1.0);
};
```

### 6.3 degradationService.ts

退化処理ロジック。

```typescript
export interface DegradationResult {
  shouldDegrade: boolean;
  expLost: number;
  newTotalExp: number;
  newStage: number;
  daysInactive: number;
}

const INACTIVITY_THRESHOLD_DAYS = 3;
const DEGRADATION_RATE_PER_DAY = 0.05; // 5%
const MAX_STAGE_DEGRADATION = 1;

export const checkDegradation = (
  currentExp: number,
  currentStage: number,
  lastPlayedAt: Date | null
): DegradationResult => {
  if (!lastPlayedAt) {
    return {
      shouldDegrade: false,
      expLost: 0,
      newTotalExp: currentExp,
      newStage: currentStage,
      daysInactive: 0,
    };
  }

  const now = new Date();
  const daysSinceLastPlay = Math.floor(
    (now.getTime() - lastPlayedAt.getTime()) / (1000 * 60 * 60 * 24)
  );

  if (daysSinceLastPlay < INACTIVITY_THRESHOLD_DAYS) {
    return {
      shouldDegrade: false,
      expLost: 0,
      newTotalExp: currentExp,
      newStage: currentStage,
      daysInactive: daysSinceLastPlay,
    };
  }

  // 退化計算
  const inactiveDays = daysSinceLastPlay - INACTIVITY_THRESHOLD_DAYS;
  const degradationMultiplier = Math.min(
    inactiveDays * DEGRADATION_RATE_PER_DAY,
    DEGRADATION_RATE_PER_DAY * 20 // 最大100%減少まで
  );

  const expLost = Math.floor(currentExp * degradationMultiplier);
  const newTotalExp = Math.max(0, currentExp - expLost);
  const newStage = getStageByExp(newTotalExp);

  // 最大1段階までの退化制限
  const limitedStage = Math.max(newStage, currentStage - MAX_STAGE_DEGRADATION);
  const limitedExp = limitedStage < newStage
    ? EVOLUTION_STAGES[limitedStage - 1].requiredExp
    : newTotalExp;

  return {
    shouldDegrade: true,
    expLost,
    newTotalExp: limitedExp,
    newStage: limitedStage,
    daysInactive: daysSinceLastPlay,
  };
};
```

### 6.4 gameEngine.ts

ゲームロジックのコア。

```typescript
export type Judgement = 'Perfect' | 'Great' | 'Good' | 'Miss';

export interface JudgementResult {
  judgement: Judgement;
  score: number;
  combo: number;
  isComboBroken: boolean;
}

export interface Note {
  id: string;
  timing: number; // ミリ秒
  type: 'up' | 'down' | 'left' | 'right';
  judged: boolean;
}

// 判定タイミングウィンドウ（ミリ秒）
const JUDGEMENT_WINDOWS = {
  Perfect: 50,
  Great: 100,
  Good: 150,
};

// スコア配分
const JUDGEMENT_SCORES = {
  Perfect: 100,
  Great: 70,
  Good: 40,
  Miss: 0,
};

export class GameEngine {
  private currentCombo = 0;
  private totalScore = 0;

  judgeNote(noteTiming: number, hitTiming: number): JudgementResult {
    const timingDiff = Math.abs(hitTiming - noteTiming);

    let judgement: Judgement;
    if (timingDiff <= JUDGEMENT_WINDOWS.Perfect) {
      judgement = 'Perfect';
    } else if (timingDiff <= JUDGEMENT_WINDOWS.Great) {
      judgement = 'Great';
    } else if (timingDiff <= JUDGEMENT_WINDOWS.Good) {
      judgement = 'Good';
    } else {
      judgement = 'Miss';
    }

    const isComboBroken = judgement === 'Miss';
    if (isComboBroken) {
      this.currentCombo = 0;
    } else {
      this.currentCombo++;
    }

    const score = JUDGEMENT_SCORES[judgement];
    this.totalScore += score;

    return {
      judgement,
      score,
      combo: this.currentCombo,
      isComboBroken,
    };
  }

  missNote(): JudgementResult {
    this.currentCombo = 0;
    return {
      judgement: 'Miss',
      score: 0,
      combo: 0,
      isComboBroken: true,
    };
  }

  reset() {
    this.currentCombo = 0;
    this.totalScore = 0;
  }
}
```

### 6.5 motionDetector.ts

モーション検出サービス。

```typescript
import { DeviceMotion, DeviceMotionMeasurement } from 'expo-sensors';

export type MotionDirection = 'up' | 'down' | 'left' | 'right' | null;

export interface MotionEvent {
  direction: MotionDirection;
  timestamp: number;
  strength: number;
}

const MOTION_THRESHOLD = 1.5; // 加速度閾値
const DEBOUNCE_MS = 200; // デバウンス時間

export class MotionDetector {
  private subscription: any = null;
  private lastDetectionTime = 0;
  private onMotionDetected: ((event: MotionEvent) => void) | null = null;

  async start(callback: (event: MotionEvent) => void) {
    this.onMotionDetected = callback;

    const { status } = await DeviceMotion.requestPermissionsAsync();
    if (status !== 'granted') {
      throw new Error('Motion sensor permission denied');
    }

    DeviceMotion.setUpdateInterval(16); // 約60fps

    this.subscription = DeviceMotion.addListener(this.handleMotion);
  }

  stop() {
    if (this.subscription) {
      this.subscription.remove();
      this.subscription = null;
    }
    this.onMotionDetected = null;
  }

  private handleMotion = (data: DeviceMotionMeasurement) => {
    const now = Date.now();
    if (now - this.lastDetectionTime < DEBOUNCE_MS) {
      return;
    }

    const { acceleration } = data;
    if (!acceleration) return;

    const { x, y, z } = acceleration;

    // 最も強い軸を検出
    const absX = Math.abs(x);
    const absY = Math.abs(y);
    const absZ = Math.abs(z);
    const maxAccel = Math.max(absX, absY, absZ);

    if (maxAccel < MOTION_THRESHOLD) return;

    let direction: MotionDirection = null;

    if (maxAccel === absY) {
      direction = y > 0 ? 'up' : 'down';
    } else if (maxAccel === absX) {
      direction = x > 0 ? 'right' : 'left';
    }

    if (direction && this.onMotionDetected) {
      this.lastDetectionTime = now;
      this.onMotionDetected({
        direction,
        timestamp: now,
        strength: maxAccel,
      });
    }
  };
}
```

### 6.6 audioPlayer.ts

音楽再生サービス。

```typescript
import { Audio, AVPlaybackStatus } from 'expo-av';

export interface AudioPlayerCallbacks {
  onTimeUpdate?: (currentTime: number) => void;
  onFinish?: () => void;
}

export class AudioPlayer {
  private sound: Audio.Sound | null = null;
  private callbacks: AudioPlayerCallbacks = {};

  async load(audioUri: string, callbacks?: AudioPlayerCallbacks) {
    await Audio.setAudioModeAsync({
      playsInSilentModeIOS: true,
      staysActiveInBackground: false,
    });

    const { sound } = await Audio.Sound.createAsync(
      { uri: audioUri },
      { shouldPlay: false },
      this.onPlaybackStatusUpdate
    );

    this.sound = sound;
    this.callbacks = callbacks || {};
  }

  async play() {
    if (!this.sound) throw new Error('Audio not loaded');
    await this.sound.playAsync();
  }

  async pause() {
    if (!this.sound) return;
    await this.sound.pauseAsync();
  }

  async stop() {
    if (!this.sound) return;
    await this.sound.stopAsync();
    await this.sound.unloadAsync();
    this.sound = null;
  }

  async getCurrentTime(): Promise<number> {
    if (!this.sound) return 0;
    const status = await this.sound.getStatusAsync();
    if (status.isLoaded) {
      return status.positionMillis;
    }
    return 0;
  }

  private onPlaybackStatusUpdate = (status: AVPlaybackStatus) => {
    if (!status.isLoaded) return;

    if (this.callbacks.onTimeUpdate) {
      this.callbacks.onTimeUpdate(status.positionMillis);
    }

    if (status.didJustFinish && this.callbacks.onFinish) {
      this.callbacks.onFinish();
    }
  };
}
```

## 7. 静的データ定義

### 7.1 songs.ts

```typescript
export interface Song {
  id: string;
  title: string;
  artist: string;
  duration: number; // 秒
  bpm: number;
  audioFile: string; // require()で読み込むパス
  coverImage: string;
  difficulties: {
    Normal: {
      noteCount: number;
      chartFile: string;
    };
    Hard: {
      noteCount: number;
      chartFile: string;
    };
  };
}

export const SONGS: Song[] = [
  {
    id: 'song_001',
    title: 'Power Up!',
    artist: 'Muscle Beats',
    duration: 120,
    bpm: 140,
    audioFile: '../assets/audio/song_001.mp3',
    coverImage: '../assets/images/cover_001.png',
    difficulties: {
      Normal: {
        noteCount: 150,
        chartFile: '../data/charts/song_001_normal.json',
      },
      Hard: {
        noteCount: 250,
        chartFile: '../data/charts/song_001_hard.json',
      },
    },
  },
  {
    id: 'song_002',
    title: 'Gym Motivation',
    artist: 'Fit Nation',
    duration: 135,
    bpm: 150,
    audioFile: '../assets/audio/song_002.mp3',
    coverImage: '../assets/images/cover_002.png',
    difficulties: {
      Normal: {
        noteCount: 180,
        chartFile: '../data/charts/song_002_normal.json',
      },
      Hard: {
        noteCount: 280,
        chartFile: '../data/charts/song_002_hard.json',
      },
    },
  },
  {
    id: 'song_003',
    title: 'Beast Mode',
    artist: 'Iron Beats',
    duration: 150,
    bpm: 160,
    audioFile: '../assets/audio/song_003.mp3',
    coverImage: '../assets/images/cover_003.png',
    difficulties: {
      Normal: {
        noteCount: 200,
        chartFile: '../data/charts/song_003_normal.json',
      },
      Hard: {
        noteCount: 320,
        chartFile: '../data/charts/song_003_hard.json',
      },
    },
  },
];
```

### 7.2 noteCharts.ts

```typescript
export interface NoteChart {
  songId: string;
  difficulty: 'Normal' | 'Hard';
  notes: Note[];
}

// 譜面JSONファイルの形式
export interface ChartJson {
  version: '1.0';
  songId: string;
  difficulty: 'Normal' | 'Hard';
  notes: Array<{
    timing: number; // ミリ秒
    type: 'up' | 'down' | 'left' | 'right';
  }>;
}
```

譜面JSONファイル例 (`song_001_normal.json`):

```json
{
  "version": "1.0",
  "songId": "song_001",
  "difficulty": "Normal",
  "notes": [
    { "timing": 1000, "type": "up" },
    { "timing": 1500, "type": "down" },
    { "timing": 2000, "type": "left" },
    { "timing": 2500, "type": "right" },
    ...
  ]
}
```

## 8. モジュール依存関係

```
screens/
  ↓ 使用
components/ ← hooks/ ← services/
  ↓ 使用              ↓ 使用
store/ ← database/ ← types/
  ↓ 使用    ↓ 使用
services/  data/
  ↓ 使用
utils/
```

**依存ルール:**
1. 上位層から下位層への依存のみ許可（循環依存禁止）
2. `screens` は `components`, `hooks`, `store`, `navigation` のみ使用
3. `components` は `hooks`, `types` のみ使用
4. `store` は `services`, `database`, `types` を使用
5. `services` は `database`, `utils`, `types`, `data` を使用
6. `database` は `types` のみ使用
7. `hooks` は `services`, `store`, `types` を使用

## 9. 技術選定理由

### 9.1 React Native (Expo managed workflow)

**選定理由:**
- クロスプラットフォーム開発（iOS/Android同時対応）
- Expo管理下で必要なネイティブ機能（SQLite、センサー、音声）が使用可能
- OTA（Over-The-Air）アップデートで迅速なバグ修正が可能
- 開発環境構築が容易

**トレードオフ:**
- ネイティブモジュールのカスタマイズに制限
- アプリサイズが大きくなる傾向

### 9.2 TypeScript

**選定理由:**
- 型安全性による実行時エラー削減
- IDEサポートによる開発効率向上
- リファクタリングの安全性
- チーム開発時のコード品質維持

### 9.3 Zustand（状態管理）

**選定理由:**
- Redux Toolkitよりも軽量でボイラープレートが少ない
- React Contextよりもパフォーマンスが良い
- 非同期処理の統合が容易
- TypeScriptとの親和性が高い

**比較:**
- Redux: 過剰な抽象化、ボイラープレートが多い
- MobX: 学習コストが高い、magic的な挙動
- Recoil: まだ実験的なライブラリ

### 9.4 expo-sqlite

**選定理由:**
- オフラインファースト設計が可能
- トレーニング記録の永続化に最適
- SQLによる柔軟なクエリが可能
- トランザクション対応でデータ整合性確保

**比較:**
- AsyncStorage: 構造化データの管理に不向き
- Realm: 過剰な機能、学習コスト高
- Firebase Firestore: オフライン動作の制約、コスト

### 9.5 expo-sensors（DeviceMotion）

**選定理由:**
- Expoで標準サポート
- 加速度センサーへの簡単なアクセス
- クロスプラットフォームで一貫したAPI

### 9.6 expo-av（Audio）

**選定理由:**
- Expoで標準サポート
- 音楽再生の完全な制御（再生位置、コールバック）
- バックグラウンド再生対応
- iOS/Android両方で安定動作

### 9.7 react-native-reanimated

**選定理由:**
- 60FPSの滑らかなアニメーション
- ネイティブスレッドでの実行によるパフォーマンス
- ゲームプレイ画面のノーツアニメーションに最適

### 9.8 Lottie (lottie-react-native)

**選定理由:**
- After Effectsで作成したアニメーションを直接使用可能
- 軽量なJSONフォーマット
- 進化演出など複雑なアニメーションに最適

### 9.9 react-native-calendars

**選定理由:**
- カレンダーUIの実装が容易
- カスタマイズ性が高い
- トレーニング記録のマーキングに最適

### 9.10 React Navigation (Stack Navigator)

**選定理由:**
- React Nativeの標準的なナビゲーションライブラリ
- 型安全なナビゲーション
- スタックベースの画面遷移がゲームフローに適合
- ディープリンク対応

## 10. パフォーマンス最適化戦略

### 10.1 ゲームプレイ画面

- `react-native-reanimated` でノーツアニメーションをネイティブスレッドで実行
- `useMemo` / `useCallback` でコンポーネント再レンダリング最小化
- 仮想化リストは使用しない（ノーツは一括管理）

### 10.2 データベースクエリ

- インデックス設定（played_at, song_id）
- 必要なカラムのみSELECT
- トランザクション使用でバッチ処理

### 10.3 画像・音声アセット

- 画像は適切なサイズにリサイズ（@2x, @3x対応）
- 音声ファイルは適切なビットレート（128-192kbps）
- Lottieアニメーションは複雑度を制限

## 11. テスト戦略

### 11.1 単体テスト（Jest）

- `services/`: ビジネスロジックのテスト
- `utils/`: ユーティリティ関数のテスト
- `store/`: 状態管理ロジックのテスト

### 11.2 統合テスト（React Native Testing Library）

- `components/`: コンポーネントの動作テスト
- `hooks/`: カスタムフックのテスト

### 11.3 E2Eテスト（Detox）

- 曲選択からゲームプレイ、リザルトまでのフロー
- キャラクター進化フロー
- トレーニング記録表示フロー

## 12. エラーハンドリング

### 12.1 データベースエラー

- トランザクション失敗時のロールバック
- エラー時のユーザーへの適切な通知
- ログ記録（開発時はconsole、本番時はクラッシュレポート）

### 12.2 センサーエラー

- パーミッション拒否時の適切なメッセージ
- センサー利用不可時のフォールバック（タップ操作など）

### 12.3 音声再生エラー

- ファイル読み込み失敗時の再試行
- 再生エラー時のユーザーへの通知

## 13. 今後の拡張性

### 13.1 機能追加の余地

- 楽曲の追加（data/songs.ts に追加）
- 新しい判定タイプ（長押し、スライドなど）
- オンラインランキング（Firebase連携）
- ソーシャル機能（スコア共有）

### 13.2 スケーラビリティ

- 楽曲数が増えてもパフォーマンスに影響しない設計
- キャラクター進化段階の追加が容易
- 新しい難易度の追加が容易

## 14. 開発フロー

### 14.1 推奨実装順序

1. **フェーズ1: 基盤構築**
   - プロジェクト初期化、ディレクトリ構成作成
   - DB初期化（schema.ts, migrations.ts）
   - 型定義（types/）
   - 静的データ定義（data/）

2. **フェーズ2: 状態管理・サービス層**
   - Zustand Store実装（characterStore, gameStore, recordStore）
   - Repository実装（database/repositories/）
   - Core Service実装（evolutionManager, experienceCalculator）

3. **フェーズ3: UI基礎コンポーネント**
   - 共通UIコンポーネント（components/ui/）
   - キャラクター表示コンポーネント（components/character/）
   - ナビゲーション設定（navigation/）

4. **フェーズ4: ホーム・記録画面**
   - ホーム画面実装
   - トレーニング記録画面実装
   - 退化チェック統合

5. **フェーズ5: ゲームコア機能**
   - ゲームエンジン実装（gameEngine.ts）
   - モーション検出実装（motionDetector.ts）
   - 音声再生実装（audioPlayer.ts）
   - ゲームコンポーネント（components/game/）

6. **フェーズ6: ゲーム画面**
   - 曲選択画面実装
   - ゲーム準備画面実装
   - ゲームプレイ画面実装
   - リザルト画面実装

7. **フェーズ7: 統合・調整**
   - 全画面の統合テスト
   - アニメーション調整
   - パフォーマンスチューニング
   - バグ修正

### 14.2 Git戦略

- `main`: 本番リリースブランチ
- `develop`: 開発統合ブランチ
- `feature/*`: 機能開発ブランチ
- `bugfix/*`: バグ修正ブランチ

## 15. まとめ

本アーキテクチャ設計書は、Muscle Beat MVPの実装に必要なすべての詳細を提供します。各モジュールは疎結合に設計されており、テスト容易性と拡張性を確保しています。

技術スタックは、モバイルゲーム開発に適したパフォーマンスと開発効率のバランスを重視して選定されています。Zustandによる状態管理、expo-sqliteによる永続化、react-native-reanimatedによる高速アニメーションにより、ユーザー体験の高いアプリケーションを実現します。

この設計に基づいて実装を進めることで、保守性が高く、将来の機能拡張にも対応できるコードベースを構築できます。
