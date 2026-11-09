# claude-worktree-peek

AIエージェントが worktree で作業している内容を、**worktree を壊さず・エージェントも止めずに**
ローカルで確認するための Claude Code スキルと運用規約です。

## 解決すること

エージェントを並行で走らせていると、確認したいブランチを worktree が掴んでいるため、
親リポジトリで切り替えようとして弾かれます。

```
fatal: 'feat-login' is already checked out at '.../worktrees/feat-login'
```

worktree を畳めば確認できますが、エージェントの作業場が消えます。
`git switch --detach` ならブランチを掴まないので、**どちらも諦めずに済みます**。

```
worktree  HEAD ──▶ feat-login ──▶ e5f6g7h   ← エージェントは作業を続けられる
親        HEAD ─────────────────▶ e5f6g7h   ← 同じ内容を見られる
```

## 中身

| ファイル | 内容 |
|---|---|
| `skills/peek/SKILL.md` | 親を対象ブランチへ detach で切り替えるスキル |
| `rules/worktree-conventions.md` | 親は確認専用・作業は worktree 側、という運用規約 |

**2つはセットです。** スキルだけ入れても前提が満たされず動きません。
規約だけ入れると、確認のたびに手で detach することになります。

## 導入

スキルは `gh skill` でインストールできます。

```sh
gh skill install jepapt2/claude-worktree-peek peek --agent claude-code --scope user
```

規約はプロジェクトの `CLAUDE.md` に取り込んでください。

```sh
curl -sL https://raw.githubusercontent.com/jepapt2/claude-worktree-peek/main/rules/worktree-conventions.md >> CLAUDE.md
```

## 使い方

```
peek                いま作業しているブランチを見る（コミット後の取り直しもこれ）
peek feat-login     指定したブランチを見る
peek 1234           PR #1234 のブランチを見る
```

中でやっていることは4つだけです。

```
1. 親に未コミットの変更がないか確認（あれば中断する）
2. git -C <親> switch --detach <対象>
3. 依存とDBに差分があれば更新する
4. いま見ているブランチとコミットを報告する
```

**サーバーは起動しません。** 親で dev server が動いている前提で、
detach でファイルが書き換わればホットリロードが拾います。

## 前提

- 親リポジトリは確認専用。`.env`・`node_modules`・DB が揃った唯一の実行環境
- 作業用 worktree ではサーバーを起動しない
- 親では編集・コミットしない（detached でのコミットは迷子になります）

詳しくは `rules/worktree-conventions.md` を参照してください。

## 動作確認した環境

macOS / Claude Code（2026年11月時点）。
マイグレーション照合の実装例は Prisma 前提です。他ツールはテーブル名の差し替えが要ります。

## ライセンス

MIT
