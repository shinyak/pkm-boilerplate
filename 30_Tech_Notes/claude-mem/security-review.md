# claude-mem セキュリティ評価

> **対象**: `thedotmack/claude-mem` v12.3.9（plugin配下）
> **License**: AGPL-3.0
> **方式**: ローカル HTTP worker（Bun製常駐プロセス）+ SQLite + ベクトルDB + 任意でMCPサーバ
> **評価日**: 2026-04-25（最終更新: 2026-04-26）
> **想定読者**: 業務PCで claude-mem を実地検証する担当者、および利用可否を判断するチーム

---

## 1. エグゼクティブサマリ

claude-mem は **Claude Code のセッション間記憶を持続させる** ためのプラグイン。worker と呼ばれる常駐 HTTP サーバが裏で動き、ツール呼び出しを観察してDB化、LLM で要約・圧縮し、次回セッション開始時にコンテキストへ自動注入する。

### 一行判定

> **業務PCでの試用は妥当**（Claude Code 業務利用＋ソース参照が組織承認済みの場合）。**「新規リスクの追加」ではなく「既に許容している Claude Code 利用リスクの定量的拡張」** と位置付けるのが正確。
> ブラウザ経由の localhost API 攻撃は **現代ブラウザの Private Network Access (PNA) によって実用上ほぼ無効化** されており、追加で必要な能動的措置は **環境変数による送信経路封鎖** のみ。

### 評価サマリ表

| 観点 | 評価 | 一言 |
|---|---|---|
| 機能の透明性・ドキュメント | 🟢 | OSS、README充実 |
| 外部送信（コード/会話流出） | 🟢 〜 🟡 | 既定 Anthropic のみ／第三者送信は環境変数で完全停止可 |
| ローカル API（無認証） | 🟢 〜 🟡 | PNA でブラウザ経由は実質遮断／同マシン他プロセス経路は既存と同質 |
| サプライチェーン | 🟡 | curl\|bash 自動インストール、コミット済み Mach-O バイナリ 63MB |
| シェル設定の自動改変 | 🟡 | `~/.bashrc`/`~/.zshrc` に alias 追記（可逆） |
| ライセンス（AGPL-3.0） | 🟢 | 開発支援ツール用途では伝染条項は発動しない |
| 業務PC試用での新規リスク | 🟢 | 必須措置を講じれば実用上ほぼ無し |

---

## 2. 評価の前提とスコープ

### 想定利用シナリオ

- **業務PC（macOS、IT管理下）** に試験的にインストール
- 主用途は **業務プロダクトコードのリファクタ・調査・実装支援**
- リモートリポジトリへの書き込み操作は Claude にさせない方針
- メインブラウザは Chrome / Edge（最新版維持）

### 組織承認済み事項（前提）

1. Claude Code の業務利用が法務・情シス的に許容されている
2. 業務ソースコードを Claude（Anthropic）に参照させることが許容されている

### スコープ外

- claude-mem を **業務SaaSや配布物に組み込む**用途（AGPL伝染の議論が必要、本評価では非対象）
- 共有開発サーバ・複数ユーザマシンでの利用
- 完全エアギャップ環境

---

## 3. 評価軸別の所見

### A. ネットワーク通信・外部送信

#### リスニング
- worker HTTP サーバは **`127.0.0.1` バインド**（ループバック限定、外部ホストから直接到達不可）
- ポート: デフォルト `37777`（環境変数で変更可）
- 同梱の MCP サーバは stdio 経由（ネットワーク露出なし）

#### 外部 API 呼び出し

| 送信先 | 用途 | デフォルト | 送信内容 |
|---|---|---|---|
| `api.anthropic.com`（SDK経由） | observation 圧縮・要約 | **既定で動作** | プロジェクト名／プロンプト／ツール呼び出し履歴 |
| `openrouter.ai/api/v1/chat/completions` | 代替LLMプロバイダ | OFF（API key 設定で起動） | 同上 |
| `generativelanguage.googleapis.com/v1/models` | 代替LLMプロバイダ | OFF（API key 設定で起動） | 同上 |
| `api.telegram.org/bot/sendMessage` | セキュリティ通知 | OFF（BOT_TOKEN+CHAT_ID 設定で起動） | observation 概要・プロジェクト名 |
| `bun.sh/install`、`astral.sh/uv/install.sh` | Bun/uv の curl\|bash 自動インストール | 初回 Setup 時 | — |

#### 重要ポイント
- Telegram は `security_alert` タイプ検知時に通知。**`ENABLED` 既定 true** だが Token ブランクで無動作。**Token を入れた瞬間に送信が始まる** 設計なので、明示的に `ENABLED=false` を入れておくのが安全。
- worker は `CLAUDE_CODE_OAUTH_TOKEN` を読み取って Anthropic API を叩く（SDK の正規挙動）。**worker プロセスは Claude Code と同等の認証資格を保持する**。

---

### B. フック登録の挙動

`plugin/hooks/hooks.json` の登録内容：

| フック | matcher | 内容 | リスク |
|---|---|---|---|
| **Setup** | `*` | smart-install.js（curl\|bash で Bun/uv 入れる、bashrc/zshrc 改変、bun upgrade自動実行） | 🟡 |
| SessionStart | `startup\|clear\|compact` | worker起動 + コンテキスト注入 | 🟡 |
| UserPromptSubmit | （無条件） | session-init | 🟡 |
| PreToolUse | `Read` | file-context注入（プロンプトに前回情報を差し込む） | 🟡 |
| PostToolUse | `*` | 観察記録 | 🟢 |
| Stop | （無条件） | サマライズ→DB保存 | 🟢 |
| SessionEnd | （無条件） | session-complete | 🟢 |

> **Setup フックは timeout 300秒で curl\|bash を含む大きな処理を走らせる**。プラグイン初回有効化時にユーザー確認なしで動くため、組織のソフトウェア導入ポリシーとの整合確認が必要。

---

### C. コード実行・ランタイム

#### 自動インストール / シェル改変

`smart-install.js` がプラグイン有効化時に以下を実行する：

1. `curl -fsSL https://bun.sh/install | bash` で Bun 自動インストール
2. `curl -LsSf https://astral.sh/uv/install.sh | sh` で uv 自動インストール
3. **`~/.bashrc` と `~/.zshrc` に `alias claude-mem=...` を追記**
4. Windows では PowerShell プロファイルにも追記
5. `bun upgrade` を自動実行（既存 Bun を強制アップデート）
6. `bun install` で依存（express、react、handlebars 等）を取得

→ いずれも可逆だが、**事前にスナップショットを取っておかないと撤退時に何が変わったか把握できない**。

#### コミット済み Mach-O バイナリ

- `plugin/scripts/claude-mem` が **63MB の macOS arm64 バイナリ**（git にコミット済み）
- Bun コンパイル成果物と推測されるが、ビルド再現性検証は不可
- darwin 以外では実行されず JS フォールバックが動く設計
- 中身を `strings` で確認できなかったため **不透明性そのものがリスク要因**

#### 動的コード実行

- `eval` / `Function()` の直接利用は readable scripts には見つからず（bundle 部分は確認不可）

---

### D. データ収集と保存

#### 保存先（すべてローカル）

- `~/.claude-mem/claude-mem.db`（SQLite）
- `~/.claude-mem/vector-db/`（Chroma ベクトル DB）
- `~/.claude-mem/{logs,archives,backups,trash}/`

#### SQLite スキーマから判明する記録内容

```sql
sdk_sessions      : project, user_prompt, started_at, status, ...
observations      : project, text, type, title, subtitle, facts,
                    narrative, concepts, files_read, files_modified
session_summaries : request, investigated, learned, completed,
                    next_steps, files_read, files_edited, notes
user_prompts      : (専用テーブル)
```

つまり **ユーザープロンプト全文 / 読んだ・編集したファイルパス / 会話のナラティブ要約 / プロジェクト名** が永続化される。コードのファイル内容そのものは保存されないが、**ファイル名とコンセプトレベルの要約は LLM が生成して保存**する。

#### シークレットスクラブ機構なし

- API key / 秘密情報のスクラブ用前処理コードは確認できず
- ユーザーが `.env` を Read した場合、**その内容が要約に含まれて DB と LLM に流れる可能性**がある

---

### E. ファイルシステムアクセス

| 書き込み先 | 内容 |
|---|---|
| `~/.claude-mem/` | DB・ログ・アーカイブ |
| `~/.claude/` | プラグイン関連ファイル |
| **`~/.bashrc` / `~/.zshrc`** | alias 追記 |
| **`~/Documents/PowerShell/Microsoft.PowerShell_profile.ps1`**（Win） | 関数定義追記 |
| `~/.bun/`、`~/.local/bin/uv` | runtime 本体 |

`.ssh`/`.aws`/`.gnupg` 等への明示的アクセスは grep で検出されず。

---

### F. シークレット・認証情報

#### OAuth トークン

- worker は `CLAUDE_CODE_OAUTH_TOKEN` を環境変数から読んで Anthropic API 呼び出しに使う（SDK 正規挙動）
- 「allowed envs」リストに明示的に含まれているので **意図的にトークンを通している**
- → **worker プロセスは Claude Code と同等の権限を持つ常駐プロセス**

#### 環境変数フィルタ

その他の環境変数についてはホワイトリスト/ブラックリスト的なフィルタリングロジックが見える。

---

### G. MCP サーバ

- `mcp-server.cjs` は MCP SDK ベース、**stdio 経由でネットワーク露出なし**
- 内部的には worker のローカル HTTP API をラップする形態
- MCP tool 経由で **DB の全 observation 検索・取得** が可能 → claude-mem 有効時、Claude が「過去の全プロジェクトの記憶」にアクセスできる

---

### H. サプライチェーン

| 観点 | 評価 |
|---|---|
| 配布元 | 公式 marketplace（`thedotmack`）, GitHub |
| メンテナ | Alex Newman（個人開発） |
| バージョン固定 | プラグインバージョンは marketplace で固定 |
| **コミット済みバイナリ** | 🟡 63MB Mach-O をリポジトリに直接コミット |
| 依存 pin 度 | npm 依存は `^` 範囲指定（floating） |
| 自動更新 | `bun upgrade` を勝手に走らせる |
| 署名検証 | 無し |

---

### I. 依存関係

- `plugin/package.json` の dependencies は **tree-sitter 系のみ**（コードパース用）— 健全
- ルート `package.json` には：
  - `@anthropic-ai/claude-agent-sdk`（Anthropic公式）
  - `@modelcontextprotocol/sdk`（MCP公式）
  - `express`（HTTPサーバ）
  - `react`/`react-dom`（UI、ダッシュボード用と推測）
  - `handlebars`（テンプレート、過去にXSSの歴史あり）
  - `dompurify`（サニタイザ）
- typosquatting 系・怪しいパッケージは見当たらず

---

### J. プライバシー / 識別子

- マシンID・ユーザー名・ホスト名を**外部に送る痕跡は確認されず**
- プロジェクト識別はローカルディレクトリ名（外部送信時は LLM プロンプトに含まれる）
- OpenRouter/Gemini を設定した場合のみ、observation が第三者 LLM に到達

---

### K. 透明性・コントロール

| 観点 | 評価 |
|---|---|
| OSS 可読性 | 🟡 plugin scripts のうち `worker-service.cjs`（2MB）、`mcp-server.cjs`（392KB）は bundle/minify 済みで実質不透明（ソースは GitHub の `src/` に存在） |
| ドキュメント | 🟢 README 充実、多言語訳あり |
| 機能 ON/OFF 粒度 | 🟢 環境変数で多くの機能を細かく制御可能 |
| アンインストール | 🟡 プラグイン削除しても `~/.claude-mem/`、bashrc/zshrc の alias は残る |

---

### L. 失敗時の挙動

- フックは `{"continue":true,"suppressOutput":true}` を返してエラー時もツール実行を **止めない（fail-open）**
- worker 停止中は機能停止するだけ（Claude Code のワークフロー自体には影響しない）
- インストール失敗時も JSON を返してフローは止めない

→ 安全寄りの設計だが **サイレント失敗で動いていないことに気づきにくい** 面はある。

---

### M. ライセンス（AGPL-3.0）

- AGPL の伝染は「**AGPL コードを使ったソフトウェアを配布／SaaSとして公開する**」時に発動する
- claude-mem は **業務コードと機能結合しない開発支援ツール**として動くだけ
- → 業務コードと claude-mem は機能的に結合しないため、**AGPL 伝染条項は発動しない**
- → 業務PCでの試用・日常開発支援用途では **実務リスクは小さい**
- ただし claude-mem 自体を改変して再配布する場合は AGPL 義務が発生する点は念頭に

---

### N. Claude/LLM 特有のリスク

#### 間接プロンプトインジェクション
- `PreToolUse(Read)` と `SessionStart` でコンテキストを **プロンプトに注入** する設計
- 過去にユーザーが Read したファイルや会話の要約を DB から引いてプロンプトに混ぜる
- → 過去に Read した悪意あるサイト内容が **後続セッションのコンテキストとして再生される** 経路が原理的に存在
- ユーザー視点ではこの注入は不可視

#### 設定ファイル改変
- `~/.claude/CLAUDE.md` への自動書き換えは確認されず
- プラグイン自前の `plugin/CLAUDE.md` のみを使用

---

## 4. ローカル API 脆弱性の現実的脅威モデル

claude-mem worker は `127.0.0.1:37777` で **50以上の API エンドポイント** を **認証なし** で公開している。これは元レポートで「最大の懸念点」とした事項。本セクションでは **CORS／PNA／同マシンプロセス** の3層で実際の悪用可能性を整理する。

### 4.1 観察された事実

```
/api/admin/shutdown    /api/admin/restart    /api/admin/doctor
/api/observations      /api/observations/batch
/api/sessions/init     /api/sessions/summarize
/api/memory/save       /api/import
/api/search/observations /api/search/sessions
/api/context/inject    /api/context/preview
...（合計50+）
```

| 確認内容 | 結果 |
|---|---|
| `127.0.0.1` バインド | ✅（外部ネットワークから直接到達不可） |
| Authorization / token チェック | ❌ 無し |
| CORS の明示的設定 | ❌ `app.use(cors())` 呼び出し見つからず |
| Origin / Host チェック | ❌ 見当たらず |

### 4.2 CORS による保護の限界

CORS は **「JS が応答を読めるか」** を制御する仕組みであって、**「リクエストがサーバに届くか」** を制御するものではない。

| 操作 | CORS 設定なしの挙動 |
|---|---|
| GET でデータ読み出し | リクエストは届くが、JS は応答を読めない（実質 exfil 不可） |
| Simple POST（text/plain 等） | **リクエストは届きサーバ側で副作用が起こる**。応答は読めない |
| JSON POST（preflight 必要） | preflight が失敗し、リクエスト自体が送られない |

→ CORS だけでは **副作用を起こす simple POST 攻撃** は防げない。

### 4.3 Private Network Access (PNA) による保護 — 現代ブラウザでは実質的に最強の防御層

Chrome 124+（2024年〜）および Edge / Chromium 系で標準化されている **Private Network Access** が、まさに **public origin → localhost への攻撃を一括で塞ぐ** ことを目的に設計された機構。

#### PNA の動作
- ブラウザが「リクエスト元の IP アドレス空間」と「リクエスト先の IP アドレス空間」を分類
- public（インターネット）→ private/loopback への遷移を検出すると **強制 preflight (CORS-PNA)** を発火
- ターゲットサーバが `Access-Control-Allow-Private-Network: true` ヘッダを返さなければ **リクエスト自体が送信されない**（simple request も含む）
- DNS Rebinding 後でも、**ドキュメントを読み込んだ時点の network classification を保持**するので後から 127.0.0.1 を狙う攻撃も塞がれる

#### claude-mem に当てはめると
- worker は CORS / PNA ヘッダを **何も返さない**
- → **PNA 対応ブラウザからの public origin 由来のリクエストは preflight 段階で全て弾かれる**
- → simple POST 副作用攻撃も DNS Rebinding 経由のフルアクセスも **どちらも遮断**

### 4.4 残存する経路と現実的な脅威評価

| 攻撃経路 | PNA 考慮後の評価 | 業務PC での現実性 |
|---|---|---|
| 外部サイトの GET 経由データ exfil | 🟢 SOP+PNA で実質ブロック | 非実体的 |
| 外部サイトの simple POST 副作用 | 🟢 PNA preflight で送信前阻止 | 非実体的 |
| DNS Rebinding 経由 | 🟢 PNA の network classification 保持で阻止 | 非実体的 |
| **同マシン他プロセス（curl, npm postinstall等）** | 🟡 ブラウザ介さないので PNA 対象外 | **既存リスク同質**（後述） |
| 悪意ある拡張機能（localhost 権限あり） | 🟡 拡張は PNA を bypass 可能 | 業務 PC の拡張管理ポリシー次第 |
| 非対応ブラウザ（古い Firefox 等） | 🟡 状況依存 | Chrome/Edge 主体運用なら実質非問題 |
| 同 LAN 別ホスト | 🟢 `127.0.0.1` バインドで到達不可 | 非実体的 |

### 4.5 「同マシン他プロセス経由」の位置付け

ブラウザを介さず curl 等で直接 worker に話しかけるリスクは PNA の対象外だが、これは **`~/.claude/projects/*.jsonl`（Claude Code 本体のトランスクリプト）に対するアクセスと同質の脅威モデル**。Claude Code 業務利用承認時点で同種のリスクは既に組織として許容されている範囲であり、**新規リスクではなく既存リスクの定量的拡張** と整理できる。

### 4.6 結論：localhost API のリスク評価

> 業務PC（管理下、Chrome/Edge 最新、拡張管理ポリシー有り）の運用条件であれば、**localhost API 無認証問題は実用上ほぼ脅威にならない**。
> 残るのは「サプライチェーン攻撃で同マシン上に悪意ある別プロセスが落ちてきた場合」だけで、これは Claude Code 利用そのものと同等のリスククラス。

---

## 5. 業務PC試用シナリオでの結論

### 5.1 結論

**前提条件**：
1. Claude Code の業務利用が承認済み
2. 業務ソースコードを Anthropic に参照させることが承認済み
3. claude-mem を業務 SaaS や配布物に組み込まない（開発支援ツール用途）

→ 上記前提下で、claude-mem 導入は **「新規リスクの追加」ではなく「既に許容している Claude Code 利用リスクの定量的拡張」**。

### 5.2 必須措置

| # | 措置 | 内容 |
|---|---|---|
| 1 | **環境変数による送信経路封鎖**（能動的措置） | `CLAUDE_MEM_TELEGRAM_ENABLED=false`／`CLAUDE_MEM_PROVIDER=claude`／OpenRouter・Gemini API key 未設定。第三者 LLM・Telegram 送信経路をコードレベルで停止 |
| 2 | **ローカル API 経由攻撃面の管理**（受動的措置） | Chrome/Edge 最新版維持により PNA が自動適用される。加えてブラウザ拡張は試用専用プロファイルで最小化／業務 PC の拡張管理ポリシーが効いていることを確認 |

### 5.3 リスク再分類（前提条件下）

| 元レポートの懸念 | 業務PC前提下での再評価 |
|---|---|
| Anthropic API への observation 送信 | ✅ 既に承認済みリスクの定量的拡張（同一データ分類・同一送信先）。新たな承認イベントは発生しない。**コスト面のみ別途見積** |
| 第三者 LLM（OpenRouter/Gemini）送信 | ✅ 環境変数で完全停止可能（残留ゼロ） |
| Telegram 送信 | ✅ 環境変数で完全停止可能（残留ゼロ） |
| ローカル API 経由 — Web/CSRF | ✅ **PNA で実用上ほぼ遮断**（Chrome/Edge 最新版前提） |
| ローカル API 経由 — 同マシン他プロセス | 🟡 Claude Code 本体トランスクリプトと同質の既存リスク |
| シェル rc 改変 | 🟡 可逆。事前バックアップ＋撤収時 diff で完全復旧可能 |
| Bun/uv 自動インストール | 🟡 社内承認済みソフトウェアリスト要確認（手続き事項） |
| AGPL ライセンス | ✅ 業務コードと機能結合しないため伝染条項は発動せず |
| コミット済み Mach-O バイナリ | 🟡 macOS のみ実行。JS フォールバックは全プラットフォームで読める |
| 間接プロンプトインジェクション（observation 経由） | 🟡 試用範囲なら影響限定的。長期運用前に再評価推奨 |

### 5.4 運用チェックリスト

#### 試用前
- [ ] `CLAUDE_MEM_TELEGRAM_ENABLED=false` を明示設定
- [ ] OpenRouter/Gemini の API key が環境変数に未設定であることを確認
- [ ] Chrome/Edge を最新版に更新（PNA 有効版）
- [ ] ブラウザ拡張を最小化／試用専用プロファイル準備
- [ ] `~/.bashrc`、`~/.zshrc` のバックアップ取得
- [ ] 既存 Bun のバージョン記録（自動アップデートで上がるため）
- [ ] Bun/uv 自動インストールと社内承認済みソフトウェアリストの整合確認
- [ ] `~/.claude-mem/claude-mem.db` が企業バックアップ対象範囲か確認

#### 試用中
- [ ] `.env` / `secrets/*` / `credentials/*` 等を Claude に Read させない
- [ ] 顧客データ・個人情報を含むファイルを参照させない
- [ ] 初日は `~/.claude-mem/logs/worker-*.log` を眺めて挙動を把握
- [ ] DB サイズの増加ペース観察（`ls -lh ~/.claude-mem/claude-mem.db`）
- [ ] DB 内容の抜き打ち確認（`sqlite3 ~/.claude-mem/claude-mem.db "SELECT type, title, subtitle FROM observations LIMIT 50;"`）

#### 撤退時
- [ ] `~/.claude-mem/` 全体の内容を最終確認後に削除
- [ ] `~/.bashrc`、`~/.zshrc` から `alias claude-mem=...` 行を削除（diff で確認）
- [ ] `lsof -iTCP:37777 -sTCP:LISTEN` で何も listen していないことを確認
- [ ] 必要に応じて Bun を試用前バージョンに戻す

### 5.5 即時撤退の判定基準

以下を観測したら試用を中止し `~/.claude-mem/` を削除すること：

1. DB の `observations.text` に **API キー・パスワード・顧客情報** が含まれている
2. `~/.bashrc` `~/.zshrc` に **alias 以外の改変** がある
3. **環境変数で OFF にしたはずの Telegram / OpenRouter / Gemini への送信痕跡**
4. **既知の業務プロセス**と worker のポート競合
5. Anthropic API のコストが想定の **3倍** を超える

---

## 6. 想定外シナリオへの注意

本評価は **業務PC試用** に最適化している。以下のケースは別評価が必要：

| シナリオ | 注意点 |
|---|---|
| **共有開発サーバ** | localhost API が他ユーザに見える可能性。本評価対象外 |
| **claude-mem を業務 SaaS に組み込む** | AGPL 伝染条項が発動。本評価対象外 |
| **古い Firefox / 特殊ブラウザ環境** | PNA 非対応のためブラウザ経由攻撃面が広がる |
| **長期本格運用（数ヶ月〜年単位）** | 間接プロンプトインジェクション、DB 肥大化、コスト累積など再評価項目が増える |
| **完全エアギャップ環境** | curl\|bash と npx 経由の依存取得ができないため初回起動不可 |

---

## 7. 関連ドキュメント

- `vs-ecc-comparison.md` — ECC との利用者目線比較
- `trial-precautions.md` — 業務PC試用の詳細実施手順チェックリスト
- `ecc-security-review.md` — ECC 単体のセキュリティ評価

---

## まとめ

claude-mem は **「悪意は無いが侵襲性は高い」開発支援プラグイン**。元レポートで挙げた懸念点のうち：

- **第三者 LLM・Telegram 送信** → 環境変数で完全に潰せる
- **ローカル API 無認証 / CSRF / DNS Rebinding** → 現代ブラウザの PNA が事実上カバー
- **AGPL 伝染** → 業務コードと機能結合しない用途では発動しない
- **環境改変（Bun/uv 自動インストール、shell rc 編集）** → 可逆、事前スナップショットで管理可能

→ **業務PC試用シナリオでは、必須措置1点（環境変数封鎖）と運用チェック数項目で実用上十分**。元レポートの「使うなら覚悟して使う」という総合判定は **業務PCの組織承認済み前提下では過剰評価** だった、というのが本最終版の結論。
