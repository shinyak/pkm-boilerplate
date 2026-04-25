# everything-claude-code (ECC) セキュリティ評価レポート

> **対象**: `affaan-m/everything-claude-code` v1.10.0
> **License**: MIT
> **方式**: ローカル Node.js フック群 + 多数のスキル/エージェント定義 + MCP連携（公式系メイン）
> **評価日**: 2026-04-25

---

## 総合判定

> **「比較的安全寄りの大型プラグイン」**。コードはすべて素のJSで読める、shell rc を改変しない、自動 curl\|bash しない、ライセンスもMIT。**懸念点は "サイズが大きすぎる" ことと "デフォルトMCPサーバー6個が外部通信を行うこと"**。基本は信頼できるが、フック多数・スキル200超で挙動把握コストは高い。

| 軸 | 判定 |
|---|---|
| 機能の透明性・ドキュメント | 🟢 良好（素のJS、コメント充実） |
| 外部送信（コード/会話流出） | 🟢 ECCコード自体は外部通信なし／MCP経由でのみ外部到達 |
| ローカルAPIの認証 | 🟢 ローカルHTTPサーバー無し |
| サプライチェーン | 🟢 npm依存3つだけ、バイナリ無し |
| シェル設定の自動改変 | 🟢 **改変なし**（claude-memと対照的） |
| ライセンス | 🟢 **MIT**（業務利用可） |
| 表面積の広さ | 🟡 200+スキル、38エージェント、25フック |

---

## A. ネットワーク通信・外部送信

### 🟢 ECC コード本体
- `scripts/hooks/`、`scripts/lib/` 配下を grep した結果、**`fetch`/`http.request`/`axios` 等の外部通信コードは存在せず**
- 唯一の "外部" 参照は GitHub issue 番号がコメントに書かれている程度
- ローカルHTTPサーバーも持たない（claude-memと対照的）

### 🟡 デフォルト `.mcp.json`（プラグイン有効化で自動起動するMCPサーバー6個）

| サーバー | 種別 | 通信先 | デフォルト動作 |
|---|---|---|---|
| `github` | npx子プロセス | api.github.com（PAT必要） | PAT未設定なら無動作 |
| `context7` | npx子プロセス | context7.com | 起動するがクエリ時のみ通信 |
| `exa` | **HTTP MCP** | `https://mcp.exa.ai/mcp` | 起動するがクエリ時のみ通信 |
| `memory` | npx子プロセス | ローカルJSON | 完全ローカル |
| `playwright` | npx子プロセス | ローカルブラウザ | クエリ時のみ起動 |
| `sequential-thinking` | npx子プロセス | ローカル思考フロー | 完全ローカル |

→ **ユーザーが MCP tool を実際に呼ばない限り、外部にはデータは流れない**。ただし npm レジストリから 6 つの npx パッケージを毎セッション起動するため、サプライチェーン上の依存先が固定されない（バージョンpinはされている）。

### 🟢 オプション `mcp-configs/mcp-servers.json`（テンプレート）
20以上のMCPサーバーが「使用例」として記載されているが、これらは **API key プレースホルダ付きで明示的に設定するまで起動しない**。jira / firecrawl / supabase / vercel / cloudflare / browserbase / fal-ai 等。設定したら当然それぞれの先に通信が発生する。

---

## B. フック登録の挙動

`hooks/hooks.json` の登録内容（330行・25のフックエントリ）：

| イベント | 件数 | 主な内容 | リスク |
|---|---|---|---|
| **Setup** | **0** | 🟢 **無し** | — |
| PreToolUse | 8 | Bash検閲、設計品質チェック、ファクト強制ゲート、observe.sh等 | 🟡 |
| PostToolUse | 8 | quality-gate、format、observe.sh、governance-capture等 | 🟢 |
| PreCompact | 1 | 状態保存 | 🟢 |
| SessionStart | 1 | bootstrap（前回コンテキスト読み込み） | 🟡 |
| Stop | 6 | format/typecheck、console-log検出、cost-tracker、desktop-notify等 | 🟢 |
| PostToolUseFailure | 1 | MCP health check | 🟢 |
| SessionEnd | 1 | session marker | 🟢 |

**重要な観察**:
- 🟢 **Setup フックが無い** → claude-memと違い、プラグイン有効化時に勝手にコード実行・curl|bash・パッケージインストールしない
- 🟢 多くのフックは `async: true` で**ツール実行をブロックしない fail-open** 設計
- 🟢 全フックで `ECC_HOOK_PROFILE` / `ECC_DISABLED_HOOKS` 環境変数による個別無効化が可能（`scripts/lib/hook-flags.js`）
- 🟢 `plugin-hook-bootstrap.js` で **path traversal 対策** 済み（`resolveTarget` がプラグインroot外をrejectする）
- 🟡 PreToolUse の `gateguard-fact-force.js` は **最初のEdit/Writeをブロック** する仕様なので、自動化ワークフローで詰まる可能性あり（このレポート作成中も発動した）

---

## C. コード実行・ランタイム

### 🟢 自動インストール
- `install.sh` / `install.ps1` は **npm install と delegateのみ** — curl\|bash 無し、shell rc 改変無し、Bun/uv の勝手なインストール無し
- `package.json` の `postinstall` は **echo でバナーを出すだけ**（実害なし）

### 🟢 コミット済みバイナリ
- **無し**。すべて素のJavaScript（CommonJS）と一部bash
- 14,000行ほどのhook/libコードがあるが、**全部読める**

### 🟢 動的コード実行
- `eval` / `Function()` 直接利用は確認できず
- `child_process.spawnSync` の利用箇所は限定的（フック実行・PowerShell/osascript呼び出し・パッケージマネージャ検出）
- shell コマンド組み立て箇所は `scripts/lib/shell-split.js` でトークン化されており、コマンドインジェクションへの配慮あり

### 🟡 パッケージマネージャー自動実行
- フック (post-edit-format, stop-format-typecheck, quality-gate) が **`npm`/`yarn`/`pnpm`/`bun` を自動検出して走らせる**
- これは"ユーザーのプロジェクトの開発依存を呼び出す"ので、当該プロジェクトの依存（prettier/biome/tsc等）に既知の脆弱性があれば**間接的に影響を受ける**

---

## D. データ収集と保存

すべて **ローカルファイルのみ**。

| 保存先 | 内容 |
|---|---|
| `~/.claude/sessions/` | セッションごとのサマリ（ユーザーメッセージ200文字、使用ツール名、編集ファイルパス） |
| `~/.claude/homunculus/projects/<hash>/observations.jsonl` | continuous-learning-v2 の観察ログ |
| `~/.claude/homunculus/instincts/` | 学習されたinstinct YAML |
| `~/.claude/metrics/costs.jsonl` | コスト/トークン使用量メトリクス |
| `~/.claude/skills/learned/` | 進化させたスキル |
| `~/.claude/orchestration/` | 並行ワーカー状態 |

### 🟢 シークレットスクラブの観点
- session-end.js は **ユーザーメッセージ先頭200文字** のみ抽出（コードや出力本体は保存しない）
- ファイル**パス**は保存するがファイル**内容**は保存しない
- ただし、ユーザーが `.env` や API key 入りファイルをツール入力に貼った場合、その200文字には含まれる可能性あり

### 🔴 governance-capture（オプション機能）
- `ECC_GOVERNANCE_CAPTURE=1` を設定すると、**シークレット検出・ポリシー違反・承認要求イベントを記録** する
- これはローカル保存だが、有効化時はマッチしたツール入出力を保管 → 機密情報が増えるので扱い注意

---

## E. ファイルシステムアクセス

| 書き込み先 | 内容 |
|---|---|
| `~/.claude/` 配下 | セッション・メトリクス・homunculus・スキル各種 |
| `~/.claude/plugins/` 配下 | プラグイン本体（Claude Codeが管理） |
| **シェル設定 (`~/.bashrc` 等)** | 🟢 **改変なし** |
| グローバル設定 (`.gitconfig`等) | 🟢 改変なし |

`.ssh`/`.aws`/`.gnupg`/`.env` への明示的アクセスは grep でも検出されず。

`session-start-bootstrap.js` の plugin root 解決ロジックで `~/.claude/plugins/cache/` を walk する処理はあるが、**読み取り専用**で、書き込みは自身のプラグインディレクトリ内のみ。

---

## F. シークレット・認証情報

### 🟢 OAuth トークン
- ECC は **Anthropic API を直接呼ぶコードを持たない**（claude-memと違い、自前のworkerが claude-codeのトークンを使う作りではない）
- `CLAUDE_CODE_OAUTH_TOKEN` を読み取るコードは見当たらず

### 🟢 環境変数の扱い
- `ECC_HOOK_PROFILE`、`ECC_DISABLED_HOOKS`、`CLAUDE_PLUGIN_ROOT`、`HOME` などのみ参照
- `process.env.PATH` を子プロセス起動時に継承する程度

### 🟡 MCP サーバー経由
- `.mcp.json` の github MCP は `GITHUB_PERSONAL_ACCESS_TOKEN` を使う（ユーザー設定時）
- `mcp-configs/mcp-servers.json` のオプション群は jira / firecrawl / fal-ai / browserbase 等のAPI keyを要求 → **ユーザーが明示設定したシークレットが当該サーバープロセスに渡される**

---

## G. MCP サーバー

### 🟡 デフォルト読み込み（自動起動）
`.mcp.json` で 6 サーバー：
- 全部 `npx -y` または `http` 形式
- npx 系は **毎セッションで npm レジストリからパッケージを取得** （バージョン pin 済みなので supply-chain は安定だが、npm registry障害時に起動失敗）
- HTTP 系は **`https://mcp.exa.ai/mcp`** にクエリ時通信

### MCP の権限について
- MCP toolは Claude が呼んだ時だけ実行される
- ECCは自身ではMCP toolを使っていない（フック内で MCP は呼ばない）
- **github MCPはPATを与えるとリポジトリへの書き込みも可能**（ユーザー責任）

---

## H. サプライチェーン

| 観点 | 評価 |
|---|---|
| 配布元 | 公式marketplace、GitHub、npm |
| メンテナ | Affaan Mustafa（Anthropic hackathon winner） |
| バージョン固定 | プラグイン本体は marketplace で固定。`.mcp.json` の MCPサーバーは **正確なバージョンpin** あり（`@2025.4.8`、`@2.1.4` 等） |
| **コミット済みバイナリ** | 🟢 **無し** |
| 依存pin度 | npm依存は `^` 範囲（floating）だが、**runtime依存はわずか3つ** |
| 自動更新 | 無し |
| 署名検証 | 無し（npm標準） |

---

## I. 依存関係

`package.json` の dependencies はわずか3つ：
- `@iarna/toml` — TOMLパーサ（実績あり）
- `ajv` — JSONスキーマバリデータ（実績多数）
- `sql.js` — SQLite WASM（読み取り中心の用途）

devDependencies も eslint / typescript / c8 / markdownlint 等の **真っ当な開発ツールのみ**。

🟢 **typosquatting / 怪しいパッケージは検出されず**
🟢 **postinstall フックはバナー表示のみ**

---

## J. プライバシー / 識別子

🟢 ローカル中心：
- ユーザー名・ホスト名・マシンIDの**外部送信は確認されず**
- すべての観察データ・セッションサマリはローカルファイルのみ

🟢 外部識別子の扱い：
- continuous-learning-v2 はプロジェクトIDに **git remote URL を SHA256 でハッシュ化** して使う（remote URLが直接記録されることを防ぐ設計）
- ただしハッシュ化前の URL は内部処理で保持される

---

## K. 透明性・コントロール

| 観点 | 評価 |
|---|---|
| OSS可読性 | 🟢 **全部素のJS／Python／Bash**（minify/bundle無し） |
| ドキュメント | 🟢 README、CHANGELOG、CONTRIBUTING、多言語docs完備 |
| 機能ON/OFF粒度 | 🟢 `ECC_HOOK_PROFILE` / `ECC_DISABLED_HOOKS` で個別フック無効化可能、スキル/エージェントは選択インストール（profiles） |
| アンインストール | 🟢 `scripts/uninstall.js` あり |
| ハーネス監査 | 🟢 `scripts/harness-audit.js` で自分の構成を監査できる |

---

## L. 失敗時の挙動

- 多くのフックは `async: true` で**ツール実行をブロックしない**（fail-open）
- ただし以下は **意図的にブロック**:
  - `pre:edit-write:gateguard-fact-force` — 初回Edit/Writeをブロックして調査を強制（**このレポート作成中にも発動**）
  - `pre:bash:dispatcher` — push/設定改変系の危険コマンドを検閲
  - `pre:config-protection` — lint/format設定ファイルへの変更ブロック
- これらは「**意図的に止める設計**」であり、実は安全寄り

---

## M. ライセンス

🟢 **MIT**：
- 商用利用・社内ツール組み込み・派生物作成すべて自由
- ソース公開義務なし
- claude-memの AGPL-3.0 と対照的に**業務利用障壁なし**

---

## N. Claude/LLM 特有のリスク

### 🟡 間接プロンプトインジェクション経路
- `SessionStart` で前回セッションサマリを **プロンプトに注入**
- continuous-learning-v2 のinstinctも将来的にプロンプトに混ざる可能性
- 外部ツールから取得した内容（exa MCPの web search結果等）が**そのまま会話に流入する**ので、注入対策は本質的にはClaude側

### 🟢 設定ファイル改変
- `~/.claude/settings.json` への自動書き換えは確認されず（プラグイン側 `hooks/hooks.json` で完結）
- `CLAUDE.md` / `AGENTS.md` への自動追記は無し
- スキル経由でユーザーが明示的に追加するパターンはあり得るが、ECC自身は触らない

### 🟡 表面積の広さ
- **200以上のスキル、38のエージェント、20以上のコマンド**
- 各スキル/エージェントは独自の指示文を持ち、Claude の振る舞いを変える
- ユーザーが "全部入り" でインストールすると、**どのスキルが何時発動するか把握しきれない**
- → 信頼できる開発者のキュレーションだが、「一個の意思決定を全部委ねる」サイズではある

---

## 推奨運用ガイド

### 🟢 こういう環境で使ってOK
- 個人〜チーム開発（**業務利用OK**、MIT）
- TDD・コードレビュー・セキュリティチェックを自動化したいプロジェクト
- 信頼できるCLI開発スタイルを構築したい中・上級者

### 🟡 これだけは把握しておく
- **デフォルト6つのMCPサーバーが起動**することを把握（`.mcp.json` で確認・無効化可能）
- **gateguard-fact-force** が初回Edit/Writeで発動 → 自動化スクリプトと干渉する可能性
- continuous-learning-v2 の observer は OFF（手動 `/evolve` 推奨）
- 大量スキルのうち実際に必要なものは **install profile** で絞ったほうが認知負荷が下がる

### 🔴 こういう場面では注意
- **完全エアギャップ環境** — npx でMCPサーバー取得が必要なため、ネット無しでは初回起動不可
- **超機密プロジェクト** — exa MCP / context7 MCP が起動するだけでサーバープロセスが上がる（クエリしなければ通信ゼロだが、プロセス自体は動く）

---

## 比較サマリ：claude-mem vs ECC

| 観点 | claude-mem | ECC |
|---|---|---|
| ライセンス | AGPL-3.0 🟡 | **MIT** 🟢 |
| Setupフック自動実行 | あり（curl\|bash） 🔴 | **無し** 🟢 |
| シェルrc改変 | あり（bashrc/zshrc） 🔴 | **無し** 🟢 |
| コミット済みバイナリ | 63MB Mach-O 🔴 | **無し** 🟢 |
| ローカルHTTPサーバー | 認証無しで50+API 🔴 | **無し** 🟢 |
| 外部API送信 | Anthropic（OAuth流用）／オプションでOpenRouter/Gemini/Telegram 🟡 | **本体は無し**／MCP経由のみ 🟢 |
| 依存数 | 多数（express, react, handlebars等） 🟡 | **3つのみ** 🟢 |
| データ保存範囲 | ユーザープロンプト全文・要約 🟡 | サマリ（200文字）・ファイルパスのみ 🟢 |
| 表面積（フック/スキル数） | 中規模 | **大規模**（200+スキル） 🟡 |
| OSS可読性 | bundle/minify済み 🟡 | 全部素のJS 🟢 |

---

## まとめ

**ECC は claude-mem に比べて、明らかに "ユーザー環境を汚さない設計" になっている**。MIT ライセンスで業務にも持ち込めるし、外部送信も無い、コードは全部読める。

懸念点があるとすれば **「あまりに多機能で全体把握が難しい」** という規模感の問題のみ。これは設計上の悪意ではなくキュレーションの幅広さの結果。

「**主要な開発支援フックを最小構成で**」入れたいなら、install profile で `minimal` または `standard` に絞るのが推奨パターン。
