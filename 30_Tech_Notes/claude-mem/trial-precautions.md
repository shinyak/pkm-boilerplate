# claude-mem 業務PC実地検証 — 担当者向け注意点

> **前提シナリオ**:
> - 業務PCのClaude Code で claude-mem を **試用** する
> - **業務のプロダクトコードを Claude に参照させる**（Read される）
> - **リモートリポジトリへのアクセスはさせない**（git push 等を Claude にやらせない方針）
>
> **このドキュメントの目的**: 単体セキュリティレビュー (`claude-mem-security-review.md`) を、業務PC試用という具体シナリオに落として「事前・試用中・試用後」のチェックリストにする。

---

## ⚠️ 重要な前提の認識合わせ

### 「リモートリポジトリにアクセスしない」の **罠**

この方針は **Claude にgit pushさせない** という意味だと思うけど、**claude-mem 自身はリモートリポジトリとは別の経路で外部通信する**わよ。区別しておいて：

| 通信経路 | 「リモートリポジトリにアクセスしない」で防げる？ |
|---|---|
| `git push origin` を Claude が実行 | ✅ 防げる（Claudeに指示しなければOK） |
| Claude Code 本体の Anthropic API 通信 | ❌ 防げない（Claude Code利用そのもの） |
| **claude-mem worker が裏で叩く Anthropic API** | ❌ **防げない**（worker が常時叩く） |
| Telegram（オプション） | ✅ Token未設定なら無動作 |
| OpenRouter / Gemini（オプション） | ✅ API key未設定なら無動作 |

→ **claude-mem は Claude Code とは別に、独自に Anthropic API を叩いて observation を圧縮する**。プロダクトコード内容（ファイル名、ナラティブ要約、コンセプト抽出）が **Claude Code の通常通信に上乗せして** Anthropic に送られる、ということを担当者と組織は理解しておく必要があるわ。

### 業務利用におけるライセンス上の地雷

claude-mem は **AGPL-3.0**。

- 「個人マシンで試す」ぶんには影響なし
- でも **業務PCで業務コードに対して使う** と、解釈次第で AGPL の伝染条項が議論になる場合がある（claude-mem コードがあなたの業務SaaSを"使って"いるわけではないので、通常は問題ない）
- **試用結果を社内で展開する判断材料にする前に、法務に「AGPLツールを業務開発端末で使うことは社内ポリシー上問題ないか」を一言確認** しておくと安心

---

## 🛑 STEP 1: インストール前に確認・準備すること

### 1-1. 組織側の確認（最優先）

- [ ] **DLP/情報漏洩防止ポリシー** に Anthropic API 送信が許容されているか確認（普段から Claude Code を使えていれば概ねOK）
- [ ] **AGPL ツールの業務端末利用** が情シス/法務的に許容されるか確認
- [ ] **`curl | bash` 形式のインストール** が情シス的に許容されるか確認（claude-mem の Setup フックは Bun/uv を curl|bash で勝手に入れる）
- [ ] **`api.telegram.org` への通信** がプロキシ/FW で遮断されている方が安全（誤発火時の保険）
- [ ] **業務PCに Bun が既に入っているか** 確認（無ければ claude-mem が勝手に入れる）

### 1-2. 試用前に物理的にやっておくこと

- [ ] **ホストFW で port 37777 をブロック**（Mac の場合）
  ```bash
  # 例：pf を使う場合（要管理権限）
  # 設定例は社内SE と相談すること
  ```
  - **理由**: claude-mem worker は認証なしで `127.0.0.1:37777` に50+APIを公開する。同マシン上の他プロセスやブラウザから無認可でアクセス可能
- [ ] **環境変数で Telegram を明示的に無効化**（デフォルトは `ENABLED=true`、Tokenブランクで無動作だが念のため）
  ```bash
  # 試用前に ~/.claude-mem/settings.json か該当の export 設定で
  CLAUDE_MEM_TELEGRAM_ENABLED=false
  ```
- [ ] **OpenRouter/Gemini の API key 環境変数が設定されていないこと** を確認
  ```bash
  env | grep -E "(CLAUDE_MEM_OPENROUTER|CLAUDE_MEM_GEMINI|OPENROUTER_API_KEY|GEMINI_API_KEY)"
  ```
- [ ] **既存の Bun のバージョンを記録**（claude-mem が `bun upgrade` を勝手に走らせるため、後で戻せるように）
- [ ] **`~/.bashrc` と `~/.zshrc` のバックアップ**（インストール時に alias 追記される）
  ```bash
  cp ~/.bashrc ~/.bashrc.before-claude-mem 2>/dev/null
  cp ~/.zshrc ~/.zshrc.before-claude-mem 2>/dev/null
  ```

### 1-3. 業務コード側の事前確認

- [ ] **検証対象のプロジェクトに `.env` や認証情報ファイルが含まれていないか**
  - claude-mem は **シークレットスクラブ機能を持たない**
  - Claude が `.env` を Read すると、その内容が observation 経由でDBとAnthropic API に流れる可能性
- [ ] **対象プロジェクトの `.gitignore` で除外されているファイル**（IDE設定、認証情報、顧客データ等）がワーキングツリーにあるか確認
- [ ] **ブラウザを閉じる**（または信頼できないサイトを開かない）
  - **理由**: worker起動中、悪意あるWebページから `fetch('http://127.0.0.1:37777/api/admin/shutdown', {method:'POST'})` 等のCSRF的呼び出しが届く可能性

---

## 🚦 STEP 2: 試用中の運用ルール

### 2-1. claude-mem に "見せない" 運用

- [ ] **顧客データ・機密情報を含むファイル** を Claude に Read させない
  - PreToolUse(Read) フックで観察される
  - DB に narrative summary が残る
- [ ] **`.env` / `secrets/*.json` / `credentials/*` などを開かせない**
  - 開いた瞬間、claude-mem の observation に内容が乗る
- [ ] **個人情報を含む実データが入ったテストファイルを参照させない**
- [ ] **業務上重要なIPが結晶化したアルゴリズム** をワンショットで Claude に投げない
  - 圧縮要約として Anthropic に送信される
  - 通常のClaude Code利用と同レベルだが、claude-memは**追加で同じ内容を再送信する**点に注意

### 2-2. 動作監視

- [ ] **試用初日は worker のログを確認**
  ```bash
  tail -f ~/.claude-mem/logs/worker-$(date +%Y-%m-%d).log
  ```
- [ ] **DBサイズの増加ペースを把握**
  ```bash
  ls -lh ~/.claude-mem/claude-mem.db
  ```
- [ ] **`~/.claude-mem/observations.jsonl` 等を眺めて、想定外のファイル名/プロンプト断片が記録されていないか定期確認**

### 2-3. 行動制約

- [ ] **試用期間中、業務PCで** **怪しいWebサイトを開かない**（CSRF経路の遮断）
- [ ] **試用期間中、業務PCで** **見慣れない npm/pip パッケージを試さない**（postinstall 経路）
- [ ] **共有マシン**（リモートサーバへの SSH トンネル経由のアクセスを他人が持っているマシン等）では試用しない

---

## 🧹 STEP 3: 試用後の片付け（撤収）

claude-mem を**やめる場合**、プラグインの uninstall だけでは消えないものが残るわ：

- [ ] **`~/.claude-mem/` ディレクトリを削除**
  ```bash
  # ★ 業務コードの要約が含まれているので、削除前に内容を再度確認
  ls -lh ~/.claude-mem/
  rm -rf ~/.claude-mem/
  ```
- [ ] **`~/.bashrc` / `~/.zshrc` から `alias claude-mem=...` 行を削除**（バックアップとの diff で確認）
  ```bash
  diff ~/.bashrc.before-claude-mem ~/.bashrc
  diff ~/.zshrc.before-claude-mem ~/.zshrc
  ```
- [ ] **port 37777 で何も listen していないことを確認**
  ```bash
  lsof -iTCP:37777 -sTCP:LISTEN
  # 何も出力されなければOK
  ```
- [ ] **Bun が `bun upgrade` で勝手に上がっていたら**、必要なら元バージョンに戻す
- [ ] **`~/.bun/`、`~/.local/bin/uv`** を試用以前から持っていなければ、必要に応じて削除（ただし他のツールが使っている可能性もあるので慎重に）
- [ ] **試用期間中に Anthropic に送られたデータの取り扱いは Anthropic 側のポリシーに依存** → 撤収時には取り戻せないことを認識（だから事前確認が大事）

---

## 📋 STEP 4: 試用評価の観点（ビジネス判断のため）

担当者が試用結果をレポートするときに、以下の観点を含めると組織判断しやすいわ：

### 機能面
- [ ] 復帰時にコンテキストが本当に有効に再現されるか
- [ ] semantic search で「過去のあれ」を引き出せたか（`/api/search/observations` 等）
- [ ] DBサイズが業務利用に耐えるか（数ヶ月で何GBになるか）
- [ ] 試用期間中の **Anthropic API 追加コスト** がいくらだったか（worker による追加呼び出し）

### セキュリティ面（試用で実証すべき）
- [ ] worker停止中も Claude Code が問題なく動くか（fail-open設計の確認）
- [ ] `~/.bashrc` に追加された alias が想定通りか
- [ ] DB に **想定外の機密情報** が記録されていないか（試用後の DB 内容を抜き打ちで確認）
  ```bash
  # 例: SQLite で observations テーブルを覗く
  sqlite3 ~/.claude-mem/claude-mem.db "SELECT type, title, subtitle FROM observations LIMIT 50;"
  ```

### 撤退判断の閾値
- [ ] DB に機密情報が混入していたら → **即時停止・DB削除・関係者報告**
- [ ] worker が他の業務ツールとポート競合した → 撤退検討
- [ ] Anthropic API のコストが想定の3倍を超えた → 撤退検討
- [ ] 業務上知られたくないファイルパスが observation に出た → スコープ縮小か撤退

---

## 🎯 ミニマル運用版（「とりあえず軽く試したい」場合の推奨設定）

```bash
# 1. 試用前にこれらをセット（業務PCの環境変数 or ~/.claude-mem/settings.json）
export CLAUDE_MEM_TELEGRAM_ENABLED=false       # 念のため明示的にOFF
export CLAUDE_MEM_PROVIDER=claude              # 第三者LLMを使わない
unset CLAUDE_MEM_OPENROUTER_API_KEY            # 念のため
unset CLAUDE_MEM_GEMINI_API_KEY                # 念のため

# 2. 試用は機密度の低いリポジトリから始める
#    （いきなりメインプロダクトではなく、サンプルや個人検証用から）

# 3. 1セッションだけ動かして、すぐ DB を覗いて「何が記録されたか」を確認
sqlite3 ~/.claude-mem/claude-mem.db ".tables"
sqlite3 ~/.claude-mem/claude-mem.db "SELECT * FROM session_summaries LIMIT 5;"

# 4. 違和感があれば即停止
bun ~/.claude/plugins/marketplaces/thedotmack/plugin/scripts/worker-cli.js stop
```

---

## 🚩 「即時撤退」の判定基準

以下を見つけたら、迷わず試用を中止して `~/.claude-mem/` を消すこと：

1. **DB の `observations.text` に API キー・パスワード・顧客情報が含まれている**
2. **`~/.bashrc` `~/.zshrc` に alias 以外の改変がある**
3. **port 37777 への外部からのアクセス試行をログで観測**（FWが正しく機能していない兆候）
4. **既知の業務プロセス**（社内開発ツール等）と worker がポート競合
5. **Telegram / OpenRouter / Gemini への送信痕跡**（環境変数で OFF のはずなのに）

---

## まとめ：試用を「ちゃんと終わらせる」ために

claude-mem は **使ってる間に勝手に環境を改変するタイプ**だから、試用は次の3点で締めくくれば安全よ：

1. **入れる前に bashrc/zshrc/Bunバージョンのスナップショットを取る**
2. **試用後に DB の中身を必ず一度は覗いて、想定外データを目視確認**
3. **撤退時は `~/.claude-mem/` ＋ shell rc ＋ port状態の3点セットを掃除**

この3つを守れば、「試したけど辞める」も「気に入ったから本格利用検討に進む」も、安全に判断できる土俵に立てるわ🩷

---

> **参考レポート**:
> - 単体評価: `~/Downloads/claude-mem-security-review.md`
> - ECC比較: `~/Downloads/claude-mem-vs-ecc-comparison.md`
