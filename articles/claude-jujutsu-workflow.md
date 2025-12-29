---
title: "Claude Code × jujutsu - AIがバージョン管理を実行する時代"
emoji: "🤖"
type: "tech"
topics: ["jujutsu", "claudecode", "ai", "versioncontrol", "git"]
published: false
---

## はじめに

> 業務自動化を専門とするPythonエンジニア。バイブコーディング歴 = エンジニア歴。
> 過去に`git stash`でデータを消失した経験あり。

「`git add`して`git commit`して...」

この呪文を何度唱えてきただろうか。そして何度、`git stash`したまま忘れて、気づいたら消えていたことか。

私は現在、**84のプロジェクトを含むモノレポ**をClaude Code（AIコーディングアシスタント）と共同で開発している。日々の作業でコミットやプッシュを行うのは**人間の私ではなく、Claude**だ。

この記事では、次世代バージョン管理システム**jujutsu（jj）**をClaude Codeと組み合わせて使う実践的なワークフローを紹介する。

## なぜjujutsuなのか

### Gitの問題点（AI開発において）

AIにGitを使わせると、以下の問題が起きやすい：

1. **ステージングの概念が複雑** - `git add`を忘れる、部分的なaddでミスる
2. **取り消しが怖い** - `git reset --hard`の破壊力
3. **コンフリクト解決が面倒** - 手動での解決が必須

### jujutsuの解決策

```bash
# Gitの場合（2ステップ）
git add .
git commit -m "feat: 新機能"

# jujutsuの場合（1ステップ、変更は自動追跡）
jj describe -m "feat: 新機能"
```

jujutsuは：
- **`git add`が不要** - ファイル変更は自動で追跡される
- **`jj undo`で何でも戻せる** - 失敗を恐れずに操作できる
- **Git完全互換** - `.git`フォルダと共存、既存のGitリポジトリで使える

## Claude Codeにjjを教える

### ルールファイルの設定

Claude Codeは`~/.claude/rules/`にあるマークダウンファイルを読み込み、指示に従う。

**~/.claude/rules/jj.md**
```markdown
# Jujutsu (jj) Rules

## 基本
- **gitコマンドは使わない** → `jj`を使用
- コミットメッセージ: Conventional Commits形式

## 主要コマンド
jj status              # 状態確認
jj diff                # 差分表示
jj describe -m "msg"   # コミットメッセージ設定
jj new                 # 新しい変更セット作成
jj bookmark set main -r @  # ブックマーク設定
jj git push            # リモートにプッシュ
```

これだけで、Claudeは「gitではなくjjを使う」ことを理解する。

### CLAUDE.mdでの強制

プロジェクトルートの`CLAUDE.md`にも明記：

```markdown
## 基本ルール
- **バージョン管理はJujutsu (jj)** を使用（gitコマンドは使わない）
- **コミットはConventional Commits**形式（`feat:`, `fix:`, `chore:`等）
```

## 実践的なエイリアス設定

### ~/.jjconfig.toml

```toml
[aliases]
# fetch + rebase を一発で（よく使う）
pull = ["util", "exec", "--", "sh", "-c", "jj git fetch && jj rebase -d main"]

# 現在の状態確認
st = ["status"]
l = ["log", "-n", "10"]

# mainから新しい作業開始
fresh = ["new", "main"]
```

### 日常のワークフロー

私がClaudeに「この修正をコミットしてプッシュして」と言うと：

```bash
# 1. Claudeが状態確認
jj status

# 2. コミットメッセージ設定
jj describe -m "fix: ブランディングページの表示条件を修正"

# 3. ブックマーク移動（mainブランチ相当）
jj bookmark set main -r @

# 4. プッシュ
jj git push
```

これらをClaudeが自動で実行してくれる。

## 84プロジェクトのモノレポ運用

### なぜモノレポ？

- 共通ライブラリの共有が容易
- 一括検索・一括置換が楽
- CI/CDパイプラインの統一

### jjでのモノレポ管理

```
amu-tazawa-scripts/
├── .jj/                    # jujutsu管理
├── kakuduke/               # プロジェクト1
├── slot_selection/         # プロジェクト2
├── pachi_chatbot/          # プロジェクト3
├── ai-secretary/           # プロジェクト4
└── ... (80+ more projects)
```

jjの利点：
- 巨大リポジトリでも高速
- 変更追跡が自動なのでプロジェクト横断の変更も楽
- 履歴操作が直感的

## MCP統合（上級編）

Claude Codeの**MCP（Model Context Protocol）**を使うと、さらに安全にjj操作ができる。

**jujutsu-expert エージェント**を設定：

```markdown
### 利用可能なMCPツール
| ツール | 用途 |
|--------|------|
| `mcp__jj__status` | 作業状態の確認 |
| `mcp__jj__describe` | コミットメッセージ設定 |
| `mcp__jj__git_push` | リモートへプッシュ |
```

MCPツール経由だと、Claudeはより構造化された形でjj操作を実行できる。

## Tips & トラブルシューティング

### 間違った操作をした場合

```bash
# 何でも戻せる！
jj undo
jj undo
jj undo  # 何度でもOK

# 操作履歴を確認
jj op log
```

### コンフリクトが発生した場合

jjはコンフリクトを「後回し」にできる。これがGitとの大きな違い。

```bash
# コンフリクトがあってもコミット可能
jj describe -m "WIP: コンフリクト解決中"

# 後でゆっくり解決
jj resolve conflicted_file.txt
```

### Claudeに教えるベストプラクティス

rulesファイルに追記：
```markdown
## ベストプラクティス
1. **失敗を恐れない**: `jj undo`で何でもやり直せる
2. **小さく頻繁に**: こまめにdescribeして履歴を残す
3. **確認してから操作**: `jj status`で状態を確認
```

## まとめ

| 観点 | Git | jujutsu |
|------|-----|---------|
| ステージング | 手動（`git add`） | 自動 |
| 取り消し | 複雑（reset, revert等） | 簡単（`jj undo`） |
| AI親和性 | 低い（ミスしやすい） | 高い（安全） |
| 学習コスト | 高い | 低い（特にGit経験者） |

**AIがコードを書く時代、バージョン管理もAIに任せる時代。**

jujutsuは「失敗しても大丈夫」という安心感を提供してくれる。Claude Codeと組み合わせることで、バージョン管理の認知負荷を大幅に減らせる。

ぜひ試してみてほしい。

---

## 参考リンク

- [jujutsu公式リポジトリ](https://github.com/martinvonz/jj)
- [Claude Code](https://claude.ai/claude-code)
- [Conventional Commits](https://www.conventionalcommits.org/)
