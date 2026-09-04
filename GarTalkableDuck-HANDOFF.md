# GarTalkableDuck 開発引き継ぎ

更新日: 2026-08-30

## 1. 目的

MicroDuck を `GaplessAgentRuntime` の Target の一つとして扱い、まず MuJoCo 上で会話と身体動作の連携を開発・検証し、後から実機 Target に差し替えられる構成を作る。

会話系は当面 **All Gemini** で構成する。音声会話とロボット制御を密結合させず、シミュレーターと実機の差を Target Adapter に閉じ込める。

## 2. リポジトリとブランチ

### 製品ブランチを持つビルド環境

- Repository: `https://github.com/ThousandsOfTies/gar-build-env.git`
- Product branch: `GarTalkableDuck`
- 現在確認済みのコミット: `cc4be37`

新しい作業ディレクトリでは、次の形で開始する。

```bash
git clone --branch GarTalkableDuck --recurse-submodules \
  https://github.com/ThousandsOfTies/gar-build-env.git
cd gar-build-env
git submodule update --init --recursive
```

### 製品固有リポジトリ

- Repository: `https://github.com/ThousandsOfTies/gar-talkable-duck.git`
- Branch: `main`
- 現在確認済みのコミット: `d479c62`
- `gar-build-env` 側の配置先: `sources/gar-talkable-duck`
- 登録済みの最小ファイル: `README.md`, `.gitignore`, `AGENTS.md`

### 共通ツール

- `sources/gar-tools`
- 現在確認済みのコミット: `aede3c5`

## 3. ブランチ運用の前提

`gar-build-env/main` は基礎部分であり、製品実装を直接入れない。製品ごとに `main` からブランチを作成し、この案件では `GarTalkableDuck` を使用する。

製品固有コードは原則として `sources/gar-talkable-duck` に置き、ビルド環境側には Target の登録、依存関係、起動構成など、統合に必要な差分だけを置く。基礎部分の更新は `gar-build-env/main` から製品ブランチへ取り込む。

初見で構造を誤解しやすいため、README では次の3点を明示する。

1. 利用者が clone するのは `gar-build-env` の製品ブランチである。
2. 製品固有実装は submodule の `gar-talkable-duck` にある。
3. 共通基盤の更新と製品固有変更では、変更先のリポジトリが異なる。

## 4. 採用する基本アーキテクチャ

```text
Microphone / Audio Input
          |
          v
Gemini realtime voice session
  - 音声認識・対話・応答生成
  - 会話イベント／感情・意図の抽出
          |                         
          +--------------------+
          |                    |
          v                    v
Speaker / Audio Output    Behavior Intent
                               |
                               v
                    Motion Coordinator
                    - 発話連動ジェスチャー
                    - 視線・姿勢・待機動作
                    - 優先度／安全制約
                               |
                         Target contract
                               |
             +-----------------+-----------------+
             |                                   |
             v                                   v
       MuJoCo Adapter                      MicroDuck Adapter
       Simulation Target                   Physical Target
```

重要なのは、Gemini にモーターを直接操作させないこと。Gemini からは `nod`, `look_at`, `happy`, `listen`, `speak_emphasis` のような意味的な Behavior Intent を出し、動作の具体化、可動域、速度制限、衝突回避は Motion Coordinator と Target 側が担当する。

## 5. MuJoCo と実機を交換可能にする境界

上位層から見える Target API を共通化する。

```text
TargetAdapter
  connect()
  disconnect()
  get_state() -> RobotState
  submit_motion(MotionCommand)
  stop_motion(reason)
  capabilities() -> TargetCapabilities
```

最低限共通化するデータ:

- joint position / velocity
- base pose（取得できる場合）
- motion lifecycle: accepted / running / completed / rejected
- actuator limits と利用可能な動作能力
- simulation time または monotonic timestamp
- emergency stop / safe pose

MuJoCo 固有の model ID、joint 名、センサー名を会話層へ漏らさない。実機固有の通信方式や SDK も `MicroDuckAdapter` の内部へ閉じ込める。これにより、設定で Target を切り替えるだけの構造を目指す。

## 6. 動作の制御権

既存の自律動作と会話連動動作は、どちらか一方がロボットを完全制御する設計にしない。Motion Coordinator が単一の actuator writer となり、複数の動作要求を調停する。

推奨する優先順位:

1. emergency stop / 安全制約
2. 転倒回避・姿勢維持など既存の低レベル制御
3. 明示的なユーザー操作
4. 会話から生成した一時的ジェスチャー
5. 既存の待機・アイドル動作

会話層は「うなずく」などの要求と期限・強度を送るだけにする。最終的な軌道と実行可否は Motion Coordinator が決める。

## 7. Gemini 方針

- 当面は音声入出力、対話、意味抽出を Gemini 系 API に統一する。
- API のモデル名はコードへ直書きせず、設定または環境変数で切り替え可能にする。
- Gemini のモデル名、Live API の提供状況、無料枠、課金条件は更新が速いため、実装開始時に公式ドキュメントで再確認する。
- ChatGPT Plus 等の消費者向けサブスクリプションは API 利用料を通常は含まないため、API 課金とは分けて扱う。
- コストを抑えるため、常時セッションではなく会話開始・無音タイムアウトを設ける。ログや映像を無条件にモデルへ連続送信しない。

API キー、プロジェクト ID、請求情報はコミットしない。`.env` は無視し、`.env.example` のみ登録する。

## 8. 最初の実装単位

最初のマイルストーンは「実機なしで一往復の音声会話と、発話に同期した MuJoCo 上のうなずきが動くこと」。

推奨順序:

1. リポジトリ内の `AGENTS.md` と既存 Target 実装を確認する。
2. `TargetAdapter` と共通データ型を定義する。
3. MuJoCo Adapter を作り、固定の MotionCommand で動作確認する。
4. Gemini 接続を実装し、音声の一往復を成立させる。
5. Gemini の会話イベントを Behavior Intent に変換する。
6. Motion Coordinator を通して発話連動ジェスチャーを再生する。
7. 通信断、API エラー、無音、割り込み、停止を試験する。
8. 実機入手後に MicroDuck Adapter を追加する。

## 9. 実装開始時の確認コマンド

```bash
git status --short --branch
git submodule status --recursive
git remote -v
git branch --show-current
```

期待値:

- current branch が `GarTalkableDuck`
- `sources/gar-talkable-duck` が初期化済み
- `sources/gar-tools` が初期化済み
- 作業開始前に意図しない変更がない

その後、リポジトリ内の指示を確認する。

```bash
find .. -name AGENTS.md -print
```

## 10. 未確定事項

実装前またはスパイク中に確定する。

- MicroDuck 実機の正式な SDK、通信方式、関節仕様、スピーカー／マイク構成
- 使用する Gemini Live モデルの正確な API model ID
- Gemini から Behavior Intent を得る方法（tool/function call または構造化イベント）
- GaplessAgentRuntime の既存 Target interface に合わせた最終 API 名
- 音声を PC 側で再生するか、MicroDuck 側へ転送するか
- MuJoCo モデルの入手元、ライセンス、実機との joint mapping

不明点を仮の実機仕様で固定せず、`TargetCapabilities` と設定ファイルで吸収する。

## 11. 新しい Codex タスクへの依頼文

以下を新しい作業ディレクトリで開いた Codex タスクの最初のメッセージとして使用できる。

> `gar-build-env` の `GarTalkableDuck` ブランチを作業対象にしてください。最初に全 `AGENTS.md`、現在のブランチ、submodule、既存 Target 実装を読み、既存設計に沿って作業計画を作ってください。製品固有実装は `sources/gar-talkable-duck`、統合差分は `gar-build-env` に置きます。会話系は All Gemini、最初の目標は MuJoCo 上で音声会話一往復と発話連動のうなずきを動かすことです。Gemini が actuator を直接操作しない構成とし、Behavior Intent、Motion Coordinator、Target Adapter の境界を保ってください。モデル ID や料金は実装時点の公式情報を確認し、秘密情報はコミットしないでください。

## 12. 現在地点

- `gar-talkable-duck/main` に最小ファイルを登録済み。
- `gar-build-env/GarTalkableDuck` を作成済み。
- `gar-talkable-duck` と `gar-tools` を submodule として登録済み。
- `--branch GarTalkableDuck --recurse-submodules` による新規 clone を確認済み。
- 本格実装、MuJoCo モデル統合、Gemini API 接続は未着手。
