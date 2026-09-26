# メモ

## 2026-09-26 [検討] Electron + TypeScript + Codex SDK によるデスクトップアプリ化

- **ステータス**: 検討中
- **対象**: `docs/claude-code-zapier-daily-work-os.md` の日常業務OS構想
- **参考**: Research-Nexus 設計書（`../Research-Nexus/docs/design.md`）の Electron 構成
- **決定事項**: エージェントは **Codex SDK（`@openai/codex-sdk`）のみ**を使う（Claude Agent SDK は使わない）

### 結論（暫定）

デスクトップアプリ化には賛成。ただし「Zapierをやめて全部デスクトップに移す」のではなく、**「Zapierは裏方、Electronアプリは操作画面」**という分け方にする。裏側では Codex SDK を呼び出す。

### Electronにした方がよい理由

今の設計は人間が確認・判断する場面が中心なので、CLIやチャットより専用の画面の方が向いている。

| 設計書の要素 | CLIでの問題 | デスクトップUIにすると |
| --- | --- | --- |
| 承認必須の操作（§9） | y/n を1件ずつ答えるしかない | 承認待ちの一覧で差分を見て、まとめて承認・却下できる |
| 朝のブリーフ（§7） | Markdownが流れていくだけ | ダッシュボードで常に表示され、項目からそのまま操作できる |
| 隙間時間の候補（§6.4） | 見に行かないと気づかない | トレイ常駐と通知で「15分空いています」と控えめに出せる |
| 余白・雑談時間の確保 | 見えにくい | タイムライン上で余白が埋まっていく様子が見える |
| 評価指標（§11） | 集計が面倒 | 修正率、削減時間、余白時間をローカルDBに残してグラフにできる |

TypeScriptでElectronを選ぶ理由：Codex SDK は Node/TypeScript 向けなので、Electronのメインプロセスから直接呼べる。Tauriだとサイドカープロセスが必要になり、手間が増える。

### Codex SDK の特性（設計に効く点）

- `new Codex()` → `startThread()` / `resumeThread(id)` → `run()` / `runStreamed()` という構成。
- 内部では `codex` CLI を子プロセスとして起動し、JSONLイベント（`thread.started`、`item.completed`、`turn.completed`、`turn.failed` など）を返す。
- `ThreadOptions` で `sandboxMode`、`approvalPolicy`、`workingDirectory`、`model` などを指定できる。
- `TurnOptions.outputSchema` で出力をJSON Schemaに固定できる。`signal`（AbortSignal）で中断できる。
- **非対話実行のため、ツール実行を途中で止めてUIに承認を求めるコールバックはない**（Claude Agent SDK の `canUseTool` に相当するものがない）。
- `resumeThread(id)` でスレッドを再開できるので、スレッドIDをSQLiteに保存すれば会話の続きをアプリ再起動後も扱える。

### 構成案

```text
Renderer (React) ─型付きIPC─ Preload ─ Main
                                          ├─ CodexRunner（@openai/codex-sdk）
                                          │    ├─ sandboxMode: read-only
                                          │    ├─ MCP: Zapier MCP（読み取り系アクションのみ公開）
                                          │    └─ outputSchema: ブリーフ / 操作提案（JSON）
                                          ├─ Approval Queue ← Codex の「操作提案」を積む
                                          ├─ Action Executor ← 承認済みの提案だけ実行（Zapier Webhook / 書き込み系API）
                                          ├─ SQLite（ブリーフ、スレッドID、実行ログ、評価指標）
                                          └─ Scheduler（アプリ起動中の定時処理）
Zapier（常時稼働）：イベント検知と夜間処理 → 結果を保存 → アプリが取り込む
```

1. **「提案」と「実行」を分ける二段階方式にする**
   Codex SDK には実行途中の承認コールバックがないため、Codex には書き込み権限を渡さない。Codex は `sandboxMode: "read-only"` と読み取り系の Zapier MCP だけで動かし、`outputSchema` で「やりたい操作」（例：`{ action: "create_gmail_draft", to, subject, body, reason }`）をJSONで返させる。承認キューで人間が承認したものだけ、アプリ側の Action Executor が実行する。設計書の allow / ask / deny は、Codex ではなく Action Executor 側で判定する。
2. **この方式は安全設計（§9）と相性がよい**
   AIが直接書き込める経路が構造的に存在しないので、「読み取りと書き込みの分離」「必要なアクションだけ公開」「冪等性キー」「入力・判断・承認・実行結果の記録」を Executor に集約できる。提案JSONに冪等性キーを付ければ二重登録も防げる。
3. **Research-Nexusと共通部品にできる**
   `ipc-contracts`、SQLiteとDrizzle、同期キュー、Electronのセキュリティ設定（contextIsolationなど）は同じ設計なので、将来は同じリポジトリ（モノレポ）にまとめられる。

### 注意点

- **アプリは常時起動していない**：PCを閉じている間の新着メール検知や朝8時の生成はできない。イベント検知と定時処理はZapier（またはサーバー側で動かす Codex）に残し、デスクトップアプリは結果を見る・承認する画面として位置付ける。
- **codex CLI バイナリの同梱**：SDK は `codex` CLI を子プロセスとして起動する。配布時にバイナリをどう同梱・更新するか（`codexPathOverride` の利用など）を決める必要がある。
- **認証**：APIキー（`CODEX_API_KEY`）で使うか、ChatGPTログインを使うかを利用規約とあわせて確認する。キーはOSの資格情報ストアに置き、Rendererには渡さない。
- **MCP設定**：Zapier MCP は Codex の設定（`~/.codex/config.toml` または SDK の `config` 指定）で登録する。アプリ専用の設定にして、普段使いの Codex 設定と混ざらないようにする。
- **機密情報の扱い**：メール本文をモデルに渡す前の検査（§9）はメインプロセスの中で実行し、Rendererには渡さない。

### MVP案

設計書の「朝のブリーフ」を次の流れで作る。

> アプリ起動 → Codex（read-only）がZapier MCP経由でメール・予定・タスクを読む → `outputSchema` でブリーフと返信下書き提案をJSONで返す → ブリーフ画面に表示 → 下書き提案を承認キューへ → 承認されたものだけ Executor が Gmail 下書きを作成

これで、UI、IPC、Codex SDK、MCP、承認フロー、Executor を一通り確認できる。

### 次のアクション（未決）

- [ ] 設計書に「デスクトップアプリ構成（Codex版）」の章を追加するか
- [ ] Electronの雛形（メイン、preload、レンダラーとCodexRunner / Action Executor）を作るか
- [ ] 提案JSON（操作提案）のスキーマを定義する
- [ ] Codex の認証方式（APIキー / ChatGPTログイン）と codex CLI の同梱方法を確定する
