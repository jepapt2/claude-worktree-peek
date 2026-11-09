---
name: peek
description: エージェントが worktree で作業中の内容を、worktree を壊さず・エージェントも止めずに親リポジトリ側で確認する。git switch --detach でブランチを掴まずにコミットだけを見るため、worktree 側が同じブランチをチェックアウトしたままでも衝突しない。「ローカルで確認したい」「このブランチ動かして見たい」「確認環境を最新にして」「peek」などの指示で発動。コミット後に表示を取り直すときも同じスキルを使う。worktree 運用規約（rules/worktree-conventions.md）とセットで使う前提で、規約なしでは前提が満たされず動かない。
argument-hint: "[ブランチ名 | PR番号]"
user-invocable: true
---

# peek — worktree の内容をローカルで確認する

## 概要

エージェントを並行で走らせていると、確認したいブランチを worktree が掴んでいるため、
親リポジトリで `git switch` しようとして弾かれる。

```
fatal: 'feat-login' is already checked out at '.../worktrees/feat-login'
```

`git switch --detach` はブランチを掴まずにコミットだけを指すので、この制限に引っかからない。
**worktree を畳む必要も、エージェントを止める必要もない。**

## 前提

- 親リポジトリは**確認専用**。ここでは編集・コミットしない
- **どのブランチにいるかは問わない。** detach するので main である必要はない
  （`peek` 実行後の detached は正常な状態。勝手に別のブランチへ戻さない）
- ただし **worktree で使う予定のブランチは掴まない**
- 親では **dev server を起動済み**であること。
  detach でファイルが書き換わればホットリロードが拾うので、このスキルでは起動しない
- 作業用 worktree ではサーバーを起動しない（`.env` や `node_modules` が無いため）

> **`rules/worktree-conventions.md` とセットで使う。**
>
> このスキルが担当するのは「確認する操作」だけ。
> 親を作業に使わない・worktree ではサーバーを起動しない・親では編集しない といった前提は規約側にある。
> **規約を入れずにこのスキルだけ導入しても、前提が満たされず期待どおりに動かない。**
> 規約はプロジェクトの `CLAUDE.md` に取り込んで使う。

## 使い方

- `peek` — いま作業しているブランチを確認する（コミット後の取り直しもこれ）
- `peek feat-login` — 指定したブランチを確認する
- `peek 1234` — PR #1234 のブランチを確認する

## ワークフロー

### Step 1: 対象ブランチの決定

- 引数が**数字** → PR番号。**`refs/pull/<N>/head` を fetch して `FETCH_HEAD` を使う**

  ```bash
  git -C "$MAIN" fetch origin "pull/$1/head"
  ```

  `gh pr view --json headRefName` でブランチ名を取るだけだと、**fork 由来のPRでは
  `origin` にそのブランチが存在せず失敗する**。`pull/<N>/head` なら fork でも解決できる。
- 引数が**文字列** → そのままブランチ名として扱う
- **省略時** → `git rev-parse --abbrev-ref HEAD`（いま作業しているブランチ）

### Step 2: 親のパスを取得

```bash
MAIN=$(git worktree list --porcelain | awk 'NR==1{print substr($0,10)}')
```

`git worktree list` の1行目が必ずメイン worktree になる。
`$2` ではなく `substr($0,10)` で取るのは、**パスに空白が含まれると `$2` が途中で切れる**ため。
`git rev-parse --show-toplevel` は**現在の worktree 自身**を返すため使わない。

### Step 3: 切り替え可能か確認する（中断条件）

次のいずれかに該当したら、**切り替えずに中断して報告する**。

1. 親に未コミットの変更がある

   ```bash
   git -C "$MAIN" status --porcelain
   ```

   **stash も破棄も勝手にしない。** ユーザーの判断を仰ぐ。

2. 親がどのブランチにも乗っていないコミットを抱えている

   ```bash
   git -C "$MAIN" branch --contains HEAD
   ```

   出力が空なら、detached のままコミットされている。
   切り替えると迷子になるので、コミット内容を提示して残すか確認する。

### Step 4: 切り替える

切り替え前のコミットを控えてから detach する。

Step 1 で決めた対象を `TARGET` に入れてから使う（ブランチ名、または PR番号なら `FETCH_HEAD`）。

```bash
BEFORE=$(git -C "$MAIN" rev-parse HEAD)
git -C "$MAIN" switch --detach "$TARGET"
```

**山括弧のプレースホルダをコマンドに直接書かないこと。** `<対象>` はシェルがリダイレクトとして
解釈するため、そのまま実行するとエラーになるか、意図しないファイルを書き換える。

ブランチ名を渡された場合は、ローカルにあれば**ローカル参照を優先する**。
無い場合だけ `git fetch origin <branch>` してから `origin/<branch>` を使う。
push 前のブランチを確認する場面が多いので、`origin/` 固定にしない。

PR番号を渡された場合は Step 1 で fetch 済みなので `FETCH_HEAD` を detach する。

### Step 5: 依存を更新する（差分があれば）

```bash
git -C "$MAIN" diff --quiet "$BEFORE" HEAD -- \
  package.json pnpm-lock.yaml package-lock.json yarn.lock
```

失敗した場合だけインストールする。pnpm なら無条件に実行しても数秒で終わる。

**プレースホルダを書かないこと。** `<lockfile>` のように山括弧で書くとシェルがリダイレクトと
解釈し、`> package.json` として**ファイルを切り詰める**。実在するファイル名を列挙する。

### Step 6: マイグレーション状態を照合する

**DBは作り直さない。** 適用履歴テーブルと、対象ブランチのマイグレーションファイル一覧を突き合わせる。

| ツール | テーブル |
|---|---|
| Prisma | `_prisma_migrations` |
| Rails | `schema_migrations` |
| Knex | `knex_migrations` |
| TypeORM | `migrations` |

```bash
# 例: Prisma（固定パスだと並行実行で衝突するので mktemp を使う）
APPLIED=$(mktemp); EXPECTED=$(mktemp)
trap 'rm -f "$APPLIED" "$EXPECTED"' EXIT

psql "$DB_URL" -tAc \
  "SELECT migration_name FROM _prisma_migrations
   WHERE finished_at IS NOT NULL AND rolled_back_at IS NULL ORDER BY 1" \
  | sed 's/ *$//' | sort > "$APPLIED"

git -C "$MAIN" ls-tree -r --name-only HEAD -- prisma/migrations \
  | awk -F/ 'NF>=3 {print $3}' | sort -u > "$EXPECTED"

comm -13 "$APPLIED" "$EXPECTED"   # 未適用
comm -23 "$APPLIED" "$EXPECTED"   # 他ブランチの残骸
```

- **未適用があれば適用する**
- **残骸があれば一覧を提示し、作り直すか確認する。** 勝手に作り直さない
- どちらもなければ「DBは一致」と報告して何もしない

gitのコミット間差分では残骸を検知できない。切り替え先の差分に現れないため、履歴テーブルと照合する必要がある。

### Step 7: 何を確認中かを記録する

後述の自動同期の判定に使う。

```bash
echo "$TARGET" > "$(git rev-parse --git-common-dir)/peek-branch"
```

`--git-common-dir` はどの worktree から実行しても共通の `.git` を指すので、別セッションからも読める。

### Step 8: 報告して終了する

```bash
git -C "$MAIN" log -1 --format='%h %s (%cr)'
```

**報告するのはブランチとコミットだけ。**

サーバーには一切触れない。親で起動済みの前提なので、起動も停止もせず、
**起動状態を報告もしない**（detach でファイルが書き換わればホットリロードが拾う）。

## 注意

- **セッションは切り替えない。** 作業しているセッションのまま呼ぶ。
  別セッションに移ると実装の経緯を知らないので説明し直しになる
- `cd` せず `git -C <path>` を使う。セッションの作業ディレクトリが動かないので迷子にならない
- detach は切り替えた時点のコミットに固定される。worktree 側が進んでも追従しないため、
  コミットのたびに呼び直す

## 補足: コミット後の自動同期

`post-commit` フックを置くと、worktree でコミットした時点で親を同期できる。
**条件を「親が detached」だけにしないこと。** 他の人が確認中の画面を奪う。

```bash
#!/usr/bin/env bash
MAIN=$(git worktree list --porcelain | awk 'NR==1{print substr($0,10)}')
COMMON=$(git rev-parse --git-common-dir)

[ "$MAIN" = "$(git rev-parse --show-toplevel)" ] && exit 0   # 親自身は対象外

# 排他ロック。取れなければ他セッションが処理中なので何もしない
LOCK="$COMMON/peek.lock"
mkdir "$LOCK" 2>/dev/null || exit 0
trap 'rmdir "$LOCK" 2>/dev/null' EXIT

# ロック取得後に判定する（判定とswitchの間に割り込まれないようにする）
git -C "$MAIN" symbolic-ref -q HEAD >/dev/null && exit 0     # 確認中ではない
CUR=$(git rev-parse --abbrev-ref HEAD)
[ "$CUR" = "$(cat "$COMMON/peek-branch" 2>/dev/null)" ] || exit 0  # 別ブランチを確認中
[ -n "$(git -C "$MAIN" status --porcelain)" ] && exit 0      # 親に未コミット変更

git -C "$MAIN" switch --quiet --detach "$CUR" || exit 0
echo "✓ 確認環境を $(git rev-parse --short HEAD) に同期しました"
```

**判定より先にロックを取ること。** 先に判定すると、`switch` までの間に別セッションが
`peek-branch` と親の HEAD を更新し、確認中の画面を奪う競合が起きる。
`mkdir` を使っているのは macOS に `flock` が無いため。

```bash
git config core.hooksPath .githooks
```

フック設定は worktree 間で共有される（`.git` が1つのため）ので、1回設定すれば全 worktree で効く。
