# pkm-boilerplate

Claude Code を「思考パートナー」として運用するための、個人向けセカンドブレイン（PKM）リポジトリの雛形。

インスパイア元: [Claude Codeを「思考パートナー」として運用する手法 - Qiita](https://qiita.com/yamapiiii/items/0ab3e12e0ec9b2bd3636)

## 何ができるか

- 曖昧な思考をテキストに書き出すと、Claude Code が分類・整理・再利用可能な形に構造化してくれる
- 過去に書いたメモやプロジェクトを横断検索して、新しい取り組みの起点に再利用できる
- 週次レビュー・ダッシュボード表示で、思考ログの健康状態を可視化できる

## 前提

- [Claude Code](https://docs.claude.com/claude-code) がインストール済みであること
- Git / GitHub CLI（`gh`）が使えること（private リポジトリ作成に使用）

## 導入手順

### 1. 雛形を取得して自分の brain リポジトリを作る

```bash
# 雛形をクローン
git clone https://github.com/<your-name>/pkm-boilerplate my-brain
cd my-brain

# 雛形の履歴を切り離す
rm -rf .git
git init

# private リポジトリとして GitHub に登録（思考ログは非公開推奨）
gh repo create my-brain --private --source=. --remote=origin
git add .
git commit -m "chore: initialize brain from pkm-boilerplate"
git push -u origin main
```

### 2. Claude Code で開く

```bash
claude
```

プロジェクトルートの `CLAUDE.md` が自動で読み込まれ、思考パートナーとして振る舞うようになる。

### 3. 別の PC で同じ brain を使う

```bash
gh repo clone <your-name>/my-brain
cd my-brain
claude
```

## ディレクトリ構造

| ディレクトリ | 用途 |
|---|---|
| `00_Inbox/` | 未整理の初期入力バッファ |
| `10_Journal/` | 日記・振り返り（`YYYY/MM/` 配下） |
| `20_Projects/` | ゴール指向のプロジェクト |
| `30_Tech_Notes/` | 永続的な技術知識 |
| `50_Business_Context/` | ビジネス・ドメイン知識 |
| `90_System/` | 運用ルール・テンプレート |
| `99_Archives/` | 完了・陳腐化したアイテム |

番号の昇順に「入力 → 熟成 → 知識化 → 卒業」の流れを表す。

## 同梱コマンド

`.claude/commands/` 配下に以下のスラッシュコマンドが定義されている。

| コマンド | 用途 |
|---|---|
| `/kickoff <テーマ>` | 新プロジェクト開始時の関連知識探索と初期構成の立ち上げ |
| `/weekly-review` | 直近 7 日間の振り返り。Inbox 未処理・停滞プロジェクト検出 |
| `/brain-status` | リポジトリの健康状態をダッシュボード表示 |

コマンドを追加したい場合は `.claude/commands/<name>.md` を新規作成する。ファイルの先頭に YAML frontmatter（`description`, `argument-hint`）を書き、本文にプロンプトを記述するだけ。

## おすすめの運用リズム

1. **毎日**: 思いついたことを Claude Code 経由で `00_Inbox/` に放り込む
2. **週次**: `/weekly-review` を実行して未処理を捌く
3. **月次**: `20_Projects/` の棚卸し、完了したものを `99_Archives/` へ
4. **随時**: 新しい関心領域が立ち上がったら `/kickoff <テーマ>` で深堀り

## カスタマイズ

- `CLAUDE.md` の「基本姿勢」「会話の姿勢」を書き換えると、Claude Code の応答トーンが変わる
- ディレクトリ構成は数字プレフィックスで順序を保つことだけ守れば、自由に増やしてよい
- `90_System/` にテンプレート（メモ雛形・プロジェクト雛形）を置いておくと、コマンドからも参照できて便利
