# claude-mem vs ECC 比較レポート（利用者目線）

> **対象**:
> - `thedotmack/claude-mem` v12.3.9
> - `affaan-m/everything-claude-code` v1.10.0
> **評価日**: 2026-04-25
> **目的**: セキュリティ・機能・問題領域を横並びで整理し、「どちらを・いつ・どう使うか」を判断する

---

## 30秒で決めるなら

```
あなたが今困っているのは…

「毎セッション同じコンテキストを再説明するのが面倒」
        → claude-mem

「TDD/レビュー/品質チェックを自動化したい」
「コーディング規約をClaudeに守らせたい」
        → ECC

「両方とも欲しい」
        → 併用OK（補完関係。ただし claude-mem 側に注意点あり）

「会社のマシン / 共有環境で使う」
        → ECC のみ推奨（claude-mem は AGPL-3.0 + 環境改変が大きい）
```

---

## 1. ポジショニング — 一行で言うと

| | claude-mem | ECC |
|---|---|---|
| **役割** | 過去セッションの**記憶**を持ち越す | 開発**ワークフロー**を標準化・高品質化 |
| **軸** | 時間軸（記憶・想起） | 品質軸（規律・ガードレール） |
| **比喩** | 開発助手の「記憶力」 | 開発助手の「育成プログラム＋作業手順書」 |
| **不在時に困ること** | 「あれ、前回どこまでやったっけ？」 | 「またテスト書き忘れた」「設定ファイル壊された」 |

---

## 2. 解決する問題領域

### claude-mem が解決する痛み

| 痛み | claude-memの提供価値 |
|---|---|
| 同じ前提説明を何度もしている | セッション開始時に過去サマリを自動注入 |
| 前回の続きから始められない | プロジェクト別に過去observationを保持・想起 |
| 過去の決定や調査結果を忘れる | semantic searchで過去の記録を検索 |
| プロジェクト横断で似た問題を解いた経験を引きたい | クロスプロジェクトの記憶検索が可能 |

### ECC が解決する痛み

| 痛み | ECCの提供価値 |
|---|---|
| Claudeが勝手に編集を始める／確認なしに動く | gateguardで初回Edit/Writeをブロックして調査強制 |
| TDD/レビュー/セキュリティの作業順序が定型化されない | `/tdd`, `/code-review`, `/security-review` 等のスキル/コマンド |
| ビルド/型エラーで止まる | `/build-fix` と言語別 build-resolver agent |
| Claudeが lint config や設定ファイルを勝手に弱める | `pre:config-protection` フックで保護 |
| 言語ごとのベストプラクティスを毎回教える | 200+のスキル（python-patterns, rust-patterns, swift-concurrency-6-2 等） |
| 開発フローを言語/スタックに合わせたい | レビュアーagentが言語別に存在（python-reviewer, go-reviewer 等） |
| プロジェクト固有の知見を蓄積したい | continuous-learning-v2（instinct system） |
| コスト感が見えない | cost-trackerで使用量メトリクス記録 |

---

## 3. 機能カバレッジ詳細比較

### A. メモリ・記憶系

| 機能 | claude-mem | ECC |
|---|---|---|
| 過去セッションの自動コンテキスト注入 | 🟢 強力（worker→DB→prompt） | 🟡 簡易（session-startで前回サマリ） |
| プロジェクト別記憶分離 | 🟢 git remote URLハッシュで分離 | 🟢 continuous-learning-v2 で同様 |
| ベクトル検索（semantic） | 🟢 Chroma DB組み込み | ❌ 無し |
| 行動パターンの抽出（instinct） | ❌ 無し | 🟢 confidence-scored instincts |
| MCP経由で記憶アクセス | 🟢 mcp-server.cjs同梱 | 🟢 memory MCP（`@modelcontextprotocol/server-memory`） |

### B. ワークフロー・規律系

| 機能 | claude-mem | ECC |
|---|---|---|
| TDDフロー強制 | ❌ | 🟢 `/tdd` + tdd-guide agent |
| コードレビュー自動化 | ❌ | 🟢 言語別レビュアー（10言語以上） |
| セキュリティレビュー | ❌ | 🟢 `/security-review` + security-reviewer agent |
| ビルドエラー解決 | ❌ | 🟢 言語別 build-resolver（C++/Go/Rust/Java/Kotlin/Dart/PyTorch） |
| 設定ファイル保護 | ❌ | 🟢 `pre:config-protection` |
| Bash検閲（`rm -rf`等） | ❌ | 🟢 `pre-bash-dispatcher` + gateguard |
| ファクト強制（編集前に調査要求） | ❌ | 🟢 gateguard-fact-force |
| 自動format/typecheck | ❌ | 🟢 言語別フック |
| console.log検出 | ❌ | 🟢 stop-check-console-log |

### C. 観察・メトリクス系

| 機能 | claude-mem | ECC |
|---|---|---|
| ツール呼び出し記録 | 🟢 全観察をDB化 | 🟢 observe.sh（軽量JSONL） |
| セッションサマリ生成 | 🟢 LLMで圧縮 | 🟢 ヘッダ／ツール名／編集ファイル抽出 |
| トークン/コスト追跡 | 🟡 限定的 | 🟢 cost-tracker.js |
| ガバナンスイベント記録 | ❌ | 🟢 governance-capture（オプション） |
| デスクトップ通知 | ❌ | 🟢 stop:desktop-notify（macOS/WSL） |

### D. MCP連携

| | claude-mem | ECC |
|---|---|---|
| 自前MCPサーバー提供 | 🟢 mcp-server.cjs | ❌ |
| デフォルト読込MCP数 | 1（自前のみ） | 6（github/context7/exa/memory/playwright/sequential-thinking） |
| テンプレートMCP設定 | 無し | 20+（jira/firecrawl/cloudflare等） |

### E. プラットフォーム展開

| | claude-mem | ECC |
|---|---|---|
| Claude Code対応 | 🟢 | 🟢 |
| Cursor対応 | 🟢 cursor-hooks/ あり | 🟢 .cursor/ 設定あり |
| OpenCode対応 | ❌ | 🟢 .opencode/ ＋ build:opencode |
| Codex対応 | ❌ | 🟢 .codex-plugin/ |
| Gemini CLI対応 | 🟡 LLMプロバイダーとして | 🟢 .gemini/ + adapt-agents |

---

## 4. セキュリティ比較（要点）

| 観点 | claude-mem | ECC |
|---|---|---|
| **ライセンス** | AGPL-3.0 🟡 | **MIT** 🟢 |
| **Setupフック** | あり（curl\|bash で Bun/uv 自動DL） 🔴 | **無し** 🟢 |
| **シェルrc改変** | `~/.bashrc`/`~/.zshrc` に alias 追記 🔴 | **無し** 🟢 |
| **コミット済みバイナリ** | 63MB Mach-O（macOS arm64） 🔴 | **無し** 🟢 |
| **ローカルHTTPサーバー** | 認証無しで50+APIを公開 🔴 | **無し** 🟢 |
| **外部送信（本体）** | Anthropic API（OAuth流用）／OpenRouter/Gemini/Telegram（オプション） 🟡 | **無し** 🟢 |
| **bundle/minify** | worker-service.cjs（2MB）等は読めない 🟡 | 全部素のJS 🟢 |
| **依存数** | express/react/handlebars等多数 🟡 | わずか3つ 🟢 |
| **保存データの粒度** | プロンプト全文＋LLM要約（narrative/facts/concepts） 🟡 | 200文字＋ファイル名のみ 🟢 |
| **業務利用の障壁** | AGPL波及リスクあり 🟡 | **無し** 🟢 |

> 👉 セキュリティ・運用負荷だけで見るなら、**ECC が圧倒的に軽い**。claude-memは機能の代償として侵襲性が高い設計。

---

## 5. インストール・運用負荷

### claude-mem

```
[初回有効化]
  ↓ Setup hook 自動実行
  - curl|bash で Bun を勝手に install
  - curl|bash で uv（Python） を勝手に install
  - bun install で express/react/handlebars等を取得
  - ~/.bashrc, ~/.zshrc に alias 追記
  ↓
[毎セッション]
  - SessionStart で worker (Bun製HTTPサーバ) 起動
  - port 37777 で常駐
  - 全ツール呼び出しを観察→DB化→LLM圧縮
  ↓
[アンインストール]
  - プラグイン削除しても ~/.claude-mem/、shell alias は残る（手動cleanup必要）
```

### ECC

```
[初回有効化]
  - 何も勝手に install しない
  - install.sh / install.ps1 を手動実行する場合のみ npm install
  ↓
[毎セッション]
  - SessionStart で前回サマリ読み込み（軽い）
  - フックは fail-open async が中心
  - .mcp.json の6サーバーが必要時に npx で起動
  ↓
[アンインストール]
  - scripts/uninstall.js があるためクリーン
```

→ claude-memは「**コンビニで弁当買ったらキッチンも建て替えられた**」感がある。ECCは「**頼んだ通りの設定だけ入れてくれた**」感じ。

---

## 6. ユーザータイプ別の推奨

| あなたの状況 | claude-mem | ECC | 併用 | コメント |
|---|---|---|---|---|
| **個人開発・趣味プロジェクト** | 🟢 | 🟢 | 🟢 | 両方の恩恵を受けやすい |
| **長期プロジェクト（数ヶ月〜年単位）** | 🟢🟢 必須級 | 🟢 | 🟢 | claude-memの真価が出る |
| **短期スクリプト・単発タスク** | ❌ 不要 | 🟢 | claude-memの記憶は無駄になる | |
| **チーム開発** | 🟡 | 🟢🟢 | 🟡 | ECCの規約強制が効く／claude-memは個人記憶 |
| **業務（受託・SaaS開発）** | ❌ AGPL注意 | 🟢🟢 | ❌ | MITのECCを推奨 |
| **共有開発サーバ・ハッカー会場** | ❌ 認証無しAPI危険 | 🟢 | ❌ | ECC一択 |
| **完全エアギャップ環境** | ❌ curl必要 | 🟡 npx要 | ❌ | どちらも要オフライン整備 |
| **TDD/品質重視** | ❌ | 🟢🟢 | claude-memは品質を上げない | |
| **多言語スタック（Go/Rust/Swift等）** | ❌ | 🟢🟢 | 言語別レビュアー/ビルドリゾルバが効く | |
| **コスト管理が必要** | ❌ 余計なAPI呼ぶ | 🟢 cost-tracker | claude-memはAPI料金を増やす | |
| **過去の意思決定を引きたい研究/設計作業** | 🟢🟢 | 🟡 | 🟢 | claude-memのsemantic searchが効く |

---

## 7. 併用する場合の注意点

両方インストールするなら以下を意識：

### ✅ 補完関係になるポイント
- ECC continuous-learning-v2 が「**行動パターン（instinct）**」を学ぶ
- claude-mem が「**何が起きたか（事実・進捗）**」を覚える
- → 役割が直交しているので衝突しない

### ⚠️ 重複・コスト面で気になるポイント
1. **両方が SessionStart でコンテキスト注入** → プロンプト先頭が二重に膨らむ
2. **両方が PostToolUse で観察** → ツール呼び出しごとに2フックが async で動く
3. **両方が Stop で要約系処理** → セッション毎に二重圧縮
4. claude-mem の worker が裏で常時 Anthropic API を叩く → **API料金増**

### 🛠 推奨する併用設定

```bash
# ECC continuous-learning-v2 の observer は OFF（手動 /evolve 推奨）
# 既にデフォルトOFF — そのまま

# claude-mem の Telegram通知は OFF のまま
# CLAUDE_MEM_TELEGRAM_ENABLED=false（デフォルトはtrueだが Tokenブランクで無動作）

# claude-mem の OpenRouter/Gemini連携は使わない
# CLAUDE_MEM_PROVIDER="claude" のままにする

# 必要なら ECC のフックを軽量プロファイルで運用
export ECC_HOOK_PROFILE=minimal
# または特定フックのみ無効化
export ECC_DISABLED_HOOKS=pre:edit-write:gateguard-fact-force
```

---

## 8. 機能 × 信頼度の二軸マッピング

```
        高機能
          ▲
          │
   claude-mem  ★（記憶系で唯一無二）
          │      …ただし侵襲性とAGPLが代償
          │
          │
          │
          │   ★ ECC
          │   （規律・品質系で総合力）
          │   …MIT・低侵襲
          │
          ├──────────────────────► 高信頼度
                                  （業務利用・共有環境向き）
```

---

## 9. 結論：用途別マトリクス

| あなたの優先順位 | 第一選択 | 補強として |
|---|---|---|
| **作業の質を上げたい** | ECC | — |
| **同じ文脈を引き継ぎたい** | claude-mem | ECC の session-end でも軽く対応可 |
| **業務コードに使いたい** | ECC（MIT） | claude-mem は社内法務確認後 |
| **個人マシンで何でも使いたい** | ECC + claude-mem 併用 | 上の推奨設定で運用 |
| **学習機能（パターン抽出）が欲しい** | ECC continuous-learning-v2 | claude-mem は記憶であって学習ではない |
| **安全側に倒したい** | ECC のみ | claude-mem は不要なら入れない |

---

## 一行まとめ

> **ECC は「Claudeの育成プログラム」、claude-mem は「Claudeの記憶装置」。**
> 困りごとが「品質」なら ECC、「記憶」なら claude-mem。両方なら併用可だが、**業務利用や共有環境では ECC 単独が安全**。
