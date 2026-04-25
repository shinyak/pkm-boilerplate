# claude-mem セキュリティ評価レポート

> **対象**: `thedotmack/claude-mem` v12.3.9（plugin配下）
> **License**: AGPL-3.0
> **方式**: ローカルHTTP worker（Bun）+ SQLite + 任意でMCPサーバー、外部LLM API利用
> **評価日**: 2026-04-25

---

## 総合判定

> **「使うなら覚悟して使う」レベル**。悪意は見えなかったけど、**システム書き換え範囲が広く、ローカルAPIに認証がない**ので、企業端末や共有マシンでは推奨しない。個人マシンで腑分けして納得した上で使うなら許容範囲。

| 軸 | 判定 |
|---|---|
| 機能の透明性・ドキュメント | 🟢 良好 |
| 外部送信（コード/会話流出） | 🟡 デフォルトはAnthropicのみ／設定でOpenRouter/Gemini/Telegram送信あり |
| ローカルAPIの認証 | 🔴 **無し**（CSRF・他プロセスから無防備） |
| サプライチェーン | 🟡 curl\|bash・npm依存・**コミット済みバイナリ63MB** |
| シェル設定の自動改変 | 🔴 `~/.bashrc`/`~/.zshrc` 自動編集 |
| ライセンス | 🟡 **AGPL-3.0**（業務利用注意） |

---

## A. ネットワーク通信・外部送信

### 🟢 リスニング
- worker の HTTP サーバーは **`127.0.0.1` バインド**（ループバック限定、外部から直接到達不可）
- ポート: デフォルト `37777`、設定可能
- 同梱の MCP サーバーも MCPプロトコル経由（stdio）

### 🟡 外部API呼び出し（用途別に整理）

| 送信先 | 用途 | デフォルト | 送信内容 |
|---|---|---|---|
| `api.anthropic.com`（SDK経由） | observation圧縮・要約 | **既定で動作** | プロジェクト名／プロンプト／ツール呼び出し履歴 |
| `openrouter.ai/api/v1/chat/completions` | 同上の代替プロバイダ | OFF（API key設定で起動） | 同上 |
| `generativelanguage.googleapis.com/v1/models` | 同上の代替プロバイダ | OFF（API key設定で起動） | 同上 |
| `api.telegram.org/bot/sendMessage` | **セキュリティ通知** | OFF（BOT_TOKEN+CHAT_ID設定で起動） | observation概要・プロジェクト名 |
| `bun.sh/install`、`astral.sh/uv/install.sh` | Bun/uv の **curl\|bash 自動インストール** | 初回Setup時 | — |

**重要ポイント**：
- Telegramは「`security_alert` タイプの観察を検知したら通知」する仕組み。**デフォルトは ENABLED=true** だけど、Tokenがブランクなら無動作。**Tokenだけセットすると即送信開始** する設計。
- `CLAUDE_CODE_OAUTH_TOKEN` を読み取って Anthropic 呼び出しに使う（SDK の正規挙動）。**worker プロセスは Claude Code の認証資格を保持する**ことになる。

---

## B. フック登録の挙動

`plugin/hooks/hooks.json` の登録内容：

| フック | matcher | 内容 | リスク |
|---|---|---|---|
| **Setup** | `*` | smart-install.js（curl\|bash で Bun/uv 入れる、bashrc/zshrc 改変、bun upgrade自動実行） | 🔴 **大** |
| SessionStart | `startup\|clear\|compact` | worker起動 + コンテキスト注入 | 🟡 |
| UserPromptSubmit | （無条件） | session-init | 🟡 |
| PreToolUse | `Read` | file-context注入（プロンプトに前回情報を差し込む） | 🟡 間接プロンプトインジェクション経路 |
| PostToolUse | `*` | 観察記録 | 🟢 |
| Stop | （無条件） | サマライズ→DB保存 | 🟢 |
| SessionEnd | （無条件） | session-complete | 🟢 |

> **Setup フックは `timeout: 300秒` で curl\|bash を含む大きな処理を走らせる**。これがプラグイン初回有効化時にユーザー確認なしで動くのが要警戒ポイント。

---

## C. コード実行・ランタイム

### 🔴 自動インストール / シェル改変

`smart-install.js` が以下を勝手にやる：
1. `curl -fsSL https://bun.sh/install | bash` で **Bunを自動インストール**
2. `curl -LsSf https://astral.sh/uv/install.sh | sh` で **uv（Python pkg manager）を自動インストール**
3. **`~/.bashrc` と `~/.zshrc` に `alias claude-mem=...` を追記**（複数シェル設定を改変）
4. Windowsでは PowerShell プロファイルにも書き込み
5. `bun upgrade` を自動実行（既存Bunを勝手にアップデート）
6. `bun install` で全依存をnpmから取得（package.json deps数も多い：tree-sitter多数、express、react、handlebars等）

これらはどれも「典型的なnpm系プラグインがやることの上限」レベル。

### 🟢 コミット済み Mach-O バイナリ
- `plugin/scripts/claude-mem` が **63MB の macOS arm64 バイナリ**（gitに直接コミット済み）
- ソースから生成された Bun コンパイル成果物っぽいが、**ビルド再現性の検証は不可**
- darwin 以外では実行されず、JS フォールバックが動く設計（macOSユーザーだけがバイナリ実行）
- 中身を `strings` で読もうとしたけど Xcode コマンドラインライセンス未承認で確認できず → 不透明性そのものがリスク

### 🟢 動的コード実行
- `eval` / `Function()` 直接利用は readable scripts には見つからず（bundleなので断定不可）

---

## D. データ収集と保存

### 保存先
- **`~/.claude-mem/claude-mem.db`** （SQLite）
- **`~/.claude-mem/vector-db/`** （ベクトル検索インデックス、Chroma DB）
- **`~/.claude-mem/logs/`**、`archives/`、`backups/`、`trash/`

### スキーマから判明する記録内容
SQLiteスキーマを直接見ると、以下を保存している：

```
sdk_sessions: project, user_prompt, started_at, status...
observations: project, text, type, title, subtitle, facts,
              narrative, concepts, files_read, files_modified
session_summaries: request, investigated, learned, completed,
                   next_steps, files_read, files_edited, notes
user_prompts: (専用テーブル)
```

つまり、
- **ユーザープロンプト全文**
- **読んだファイルパスのリスト・編集したファイルパスのリスト**
- **会話の "request"・"learned"・"completed" などのナラティブ要約**
- プロジェクト名

が永続化される。**コードのファイル内容そのものは保存されない**っぽいが、ファイル名とコンセプトレベルの要約はLLMで作られて保存される。

### 🔴 シークレットスクラブなし
- API key/secrets スクラブ用の前処理コードは確認できず
- ユーザーが `.env` 内容を Read した際、その内容が要約に含まれて DB と LLM 送信に乗る可能性

---

## E. ファイルシステムアクセス

| 書き込み先 | 内容 |
|---|---|
| `~/.claude-mem/` | DB・ログ・アーカイブ |
| `~/.claude/` | プラグイン関連ファイル |
| **`~/.bashrc` / `~/.zshrc`** | 🔴 alias 追記 |
| **`~/Documents/PowerShell/Microsoft.PowerShell_profile.ps1`**（Win） | 🔴 関数定義追記 |
| `~/.bun/`、`~/.local/bin/uv` | runtime本体（curl\|bashでインストール） |

`.ssh`/`.aws`/`.gnupg` への明示的アクセスは確認されず（grep結果で該当なし）。

---

## F. シークレット・認証情報

### 🟡 OAuth トークン
worker は `CLAUDE_CODE_OAUTH_TOKEN` を環境変数から読み取って Anthropic API 呼び出しに使う。これはプラグインを **「Claude Code と同等の権限を持つ常駐プロセス」** にすることを意味する。

明示的に「allowed envs」に入れているコードがあるので、**意図的にこのトークンを通している**。SDKの正規挙動なので即座に悪意ではないが、信頼境界の設計として留意すべき。

### 🟢 環境変数フィルタ
他環境変数についてはホワイトリスト/ブラックリスト的なフィルタリングロジックが見える。

---

## G. MCP サーバー

- 同梱の `mcp-server.cjs` は MCP SDKベース
- worker のローカル HTTP API をラップして MCP として公開する形態
- stdio ベースなのでネットワーク露出はない
- ただし MCP tools が DB の全 observation 検索・取得を提供 → **claude-memに有効化されている時、Claude が「過去の全プロジェクトの記憶」にアクセス可能**

---

## H. サプライチェーン

| 観点 | 評価 |
|---|---|
| 配布元 | 公式marketplace（`thedotmack`）, GitHub |
| メンテナ | Alex Newman（個人開発、社風依存） |
| バージョン固定 | プラグインバージョンは marketplace で固定 |
| **コミット済みバイナリ** | 🔴 **63MB Mach-O** をリポジトリに直接コミット |
| 依存pin度 | npm依存は `^` 範囲指定（floating） |
| 自動更新 | `bun upgrade` を勝手に走らせる |
| 署名検証 | 無し |

---

## I. 依存関係

`plugin/package.json` の dependencies は **tree-sitter 系のみ**（コードパース用）でそこは健全。
**ルートの `package.json` のほう**には：
- `@anthropic-ai/claude-agent-sdk`（Anthropic公式）
- `@modelcontextprotocol/sdk`（MCP公式）
- `express`（HTTPサーバー）
- `react`/`react-dom`（UI、おそらくダッシュボード用）
- `handlebars`（テンプレート、過去にXSSの歴史あり）
- `dompurify`（サニタイザ）

依存は **真っ当な顔ぶれ**。typosquatting系は見当たらず。

---

## J. プライバシー / 識別子

🟢 ローカル中心の設計：
- マシンID・ユーザー名・ホスト名を**外部に送る痕跡は確認されず**
- プロジェクト識別は **ローカルディレクトリ名** を使用（外部送信時は LLMコール時にプロンプトに含まれる可能性）

🟡 LLM送信時：
- OpenRouter/Geminiを設定している場合、observation内容（プロジェクト名、プロンプト、ツール呼び出しパターン）が **第三者LLMに到達**

---

## K. 透明性・コントロール

| 観点 | 評価 |
|---|---|
| OSS可読性 | 🟡 plugin scripts のうち `worker-service.cjs`（2MB）と `mcp-server.cjs`（392KB）は **bundle/minify済み**で実質不透明、ソースは別ディレクトリの `src/` に存在（GitHub上） |
| ドキュメント | 🟢 README充実、多言語訳もあり |
| 機能ON/OFF粒度 | 🟢 環境変数で多くの機能を細かく制御可能 |
| アンインストール | 🟡 プラグイン削除しても `~/.claude-mem/`、bashrc/zshrcの alias は残る |

---

## L. 失敗時の挙動

- フックは `{"continue":true,"suppressOutput":true}` を返してエラー時もツール実行を **止めない（fail-open）** 設計
- worker停止中は機能停止するだけ（claude-codeのワークフロー自体は影響なし）
- インストール失敗時も `process.exit(1)` だが JSON 返してフローは止めない

→ **Claude Codeを止めない設計**は安全寄りだが、**サイレント失敗** で動いてないことに気づきにくい面はある。

---

## M. ライセンス

🟡 **AGPL-3.0**：
- 個人利用は問題なし
- **業務SaaSや社内ツールに組み込むと、ソース公開義務が波及する可能性**
- AGPLコードを呼び出すサービスを公開すると関連コードも開示が必要になる場合あり
- 法務に確認推奨

---

## N. Claude/LLM 特有のリスク

### 🟡 間接プロンプトインジェクション
- `PreToolUse(Read)` と `SessionStart` でコンテキストを **プロンプトに注入** する設計
- 過去にユーザーが Read したファイルや会話の要約をDBから引いてプロンプトに混ぜる
- **悪意あるサイトの内容を Read した過去履歴がコンテキスト注入経由で再生される** 可能性
- ユーザー視点ではこの注入は不可視

### 🟢 設定ファイル改変
- `~/.claude/CLAUDE.md` への自動書き換えは確認されず
- プラグイン自前の `plugin/CLAUDE.md` のみを使用

---

## 🔴 最大の懸念点：ローカルAPIに認証なし

ここが個人的に **一番気になる** ポイント。

worker は `127.0.0.1:37777` で50以上のAPIエンドポイントを公開している：

```
/api/admin/shutdown    /api/admin/restart    /api/admin/doctor
/api/observations      /api/observations/batch
/api/sessions/init     /api/sessions/summarize
/api/memory/save       /api/import
/api/search/observations /api/search/sessions
/api/context/inject    /api/context/preview
...（合計50+）
```

調査した範囲で確認できた事実：
- ✅ `127.0.0.1` バインド（外部ネットワークから直接到達は不可）
- ❌ **Authorization ヘッダや token チェックは無い**
- ❌ **CORS は明示的に設定されていない**（同梱はされているが `app.use(cors())` 呼び出しは見つからず）
- ❌ Origin / Host チェックも見当たらず

これが意味するリスク：

1. **同一マシン内の他プロセス**（怪しい npm パッケージのpostinstall等）は `curl http://127.0.0.1:37777/api/observations` で **過去の全プロジェクトの記憶を読める**
2. **悪意ある Web ページ** を訪問した場合、ブラウザから `fetch('http://127.0.0.1:37777/api/admin/shutdown', {method:'POST'})` 等で **CSRF 的に worker を操作できる可能性**（特に Content-Type が単純なPOSTの場合）
3. **共有マシン** や開発コンテナで他ユーザーがいる場合、同じ 37777 ポートに到達される

---

## 推奨運用ガイド

### 🟢 こういう環境なら使ってOK
- 個人の単独利用マシン
- DB に入る情報をAnthropicが見てもよいと割り切れる人
- AGPL のソース公開義務を業務に持ち込まない

### 🟡 これだけは守る
- Telegram通知は **明示的に設定するまで使わない**
- OpenRouter / Gemini 連携は **本当に必要でない限り設定しない**（送信先が増える）
- インストール後 `~/.bashrc`/`~/.zshrc` に追加された alias を確認
- 業務用マシンで使う場合は法務にAGPLを確認

### 🔴 こういう環境では非推奨
- 共有開発サーバ（複数ユーザー）
- 機密プロジェクトの作業ディレクトリ
- ブラウザで開発と他作業を同居させてる端末（CSRF経路）
- AGPL を取り込めない業務環境

### 追加で気をつけたいなら
- worker 起動を **必要なときだけ** にする（環境変数 `CLAUDE_MEM_WORKER_PORT` を変えて固定ポートを避ける）
- ファイアウォール（macOS Application Firewall）で 37777 を念のためブロック
- `~/.claude-mem/claude-mem.db` のバックアップ取扱に注意（過去の全プロジェクト要約が入っている）

---

## まとめ

**悪意は見えなかった**。OSS としてまっとうに作られているし、コードも追える範囲はある。Telegramも本人の通知用、外部LLMもオプトイン。

ただし、**「ユーザーの環境を広く触る」「常駐サーバが認証なし」「コミット済みバイナリ」** という3つのパターンが揃っているので、**信頼してインストールするということは "claude-mem のメンテナとAGPLコードベースを開発環境のシステム的協力者として迎える" ことに等しい**。

claude-codeを使うのと同じくらいの信頼を一個人プロジェクトに渡せるかどうか — そこが判断のしどころ。

---

## 補遺：業務PC利用シナリオでの結論（前提条件付き再評価）

> **追記日**: 2026-04-25
> **背景**: 本評価を「業務PCでの実地検証」ユースケースに適用する際の議論の結論。
> 単独の脅威モデルではなく、**既に組織的に承認されている前提条件**の上で再評価したもの。

### 前提条件

以下が組織的に承認済みであることが前提となる：

1. **Claude Code の業務利用** が法務・情シス的に許容されている
2. **業務ソースコードを Claude（Anthropic）に参照させること** が許容されている
3. **AGPL ライセンスかす** または業務コードと機能結合しないツールとしての利用している（claude-mem は開発支援ツールとして横にいるだけで、業務SaaS・配布物の一部にならない）

### 結論

上記前提が満たされ、かつ以下2点の追加措置を講じた場合、claude-mem 導入による **新しいクラスのリスクは事実上発生しない**。

| 必須措置 | 内容 |
|---|---|
| **環境変数による送信経路封鎖** | `CLAUDE_MEM_TELEGRAM_ENABLED=false` を明示設定／`CLAUDE_MEM_PROVIDER=claude`／OpenRouter・Gemini の API key 未設定。これで第三者LLM・Telegramへの送信経路は **コードレベルで停止**（環境変数読み取り時点で early return する作りのため）。 |
| **ローカル API CSRF 対策** | worker起動中の `127.0.0.1:37777` に対する、悪意あるWebページからの `fetch` を遮断（Origin/Hostヘッダ検証 or ブラウザ動線の管理）。 |

### リスクの再分類（前提条件下）

各懸念点を「新規リスク」か「既に承認済みリスクの定量的拡張」かで区別する：

| 元レポートでの懸念 | 業務PC前提下での再評価 |
|---|---|
| Anthropic API への observation送信 | ✅ **既に承認済みリスクの拡張**。同一データ分類・同一送信先・同一処理者（Anthropic）へのトラフィック量が増えるだけで、新たな承認イベントは発生しない。コスト面は別途要見積。 |
| 第三者LLM（OpenRouter/Gemini）送信 | ✅ **環境変数で完全停止可能**（残留ゼロ）。 |
| Telegram 送信 | ✅ **環境変数で完全停止可能**（残留ゼロ）。 |
| ローカル HTTP API 認証なし — Web経由CSRF | ✅ **CSRF対策（Origin/Host検証）で遮断可能**。 |
| ローカル HTTP API 認証なし — 同マシン他プロセス経由 | 🟡 **既存リスクの定量的拡張**。`~/.claude/projects/*.jsonl` のトランスクリプトに対するアクセスと同レベルの脅威モデル。Claude Code 業務利用承認時点で、同種のリスクは既に許容されている。 |
| シェル rc 改変（bashrc/zshrc に alias 追記） | 🟡 **可逆**。事前バックアップ・撤収時 `diff` 検証で完全に元に戻せる。 |
| Bun/uv 自動インストール | 🟡 **承認済みソフトウェアリスト確認が必要なケースあり**。リスクというより手続き事項。 |
| **AGPL ライセンス** | ✅ **業務コードと機能結合しない**（claude-mem は開発支援ツールとして横にいるだけで、業務SaaS・配布物の一部にならない）ため、伝染条項は発動しない。**実務リスク小**。 |
| コミット済み Mach-O バイナリ（63MB） | 🟡 macOS でのみ実行。JSフォールバックは全プラットフォームで読める。試用範囲なら許容可。 |

### 運用上の手続き（リスクではなく "確認事項"）

「リスク」の枠組みから外して、**運用品質の確認項目** として整理すべきもの：

- [ ] Bun/uv 自動インストールと **社内承認済みソフトウェアリスト** の整合確認
- [ ] `~/.claude-mem/claude-mem.db` が **企業バックアップ対象範囲** に入るか確認（FileVault前提として、社内バックアップで業務コード要約DBが流出する設計になっていないか）
- [ ] 試用前後の **環境スナップショット** 取得・比較（`~/.bashrc`、`~/.zshrc`、Bunバージョン、port 37777状態）
- [ ] **撤退判断ライン** の事前合意（DBサイズ、APIコスト、機密情報混入時の対応）

### 「CSRF対策」の解像度メモ

「ローカルAPI CSRF対策」と言ったときの **適用範囲の限界** を担当者は認識しておくこと：

| 攻撃経路 | CSRF対策（Origin/Host検証）で防げるか |
|---|---|
| 悪意あるWebページからの `fetch('http://127.0.0.1:37777/...')` | ✅ 防げる（ブラウザがOriginヘッダを付与する） |
| **同マシンの他プロセスが直接 curl** | ❌ **防げない**（Originヘッダ無しで直接呼べる） |
| 同LANの別ホスト | ✅ そもそも `127.0.0.1` バインドで到達不可 |

→ 「同マシン他プロセス経由のリスク」は CSRF 対策の対象外。これは Claude Code 本体のトランスクリプト (`~/.claude/projects/`) に対するリスクと同質で、業務PC利用承認時点で既に許容している脅威モデルの範囲内、と整理できる。

### 総括

業務PC試用シナリオにおいて、claude-mem 導入を「**新規リスクの追加**」と捉えるのは**過剰評価**。
正しい評価軸は **「既に許容している Claude Code 利用リスクが、claude-mem 分だけ定量的に少し膨らむ」** というもの。

この軸で関係者に説明することで判断の精度が上がる。

→ **環境変数2系統の設定 + CSRF対策 + 撤収手順の事前準備** が整っていれば、業務PC試用は妥当な判断と言える。

> 関連: 業務PC試用にあたっての具体的な実施手順は `~/Downloads/claude-mem-trial-precautions.md` を参照。
