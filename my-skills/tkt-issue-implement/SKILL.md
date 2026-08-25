---
name: tkt-issue-implement
description: "「実装待」ラベルが付いた GitHub issue を順番に実装する。ブランチ作成・コード実装・docs 追記・PR 作成までを自動化し、完了後に「実装済」ラベルを付ける。最大 5 件で停止。『実装待の issue を実装して』等の依頼時に使用"
model: sonnet
---

# /tkt-issue-implement スキル

「実装待」ラベルの付いた issue を番号順に最大 5 件実装する。各 issue について、ブランチ作成・コード実装・`docs/` への設計ドキュメント追記・PR 作成をひとまとめに行い、完了後にラベルを「実装済」に切り替える。

## トリガー

以下の表現でこのスキルを使う:

- 「実装待の issue を実装して」
- 「実装を進めて」
- 「tkt-issue-implement」
- `/tkt-issue-implement tokitam/waffledb`

## 引数

- `$ARGUMENTS` に `owner/repo`（例: `tokitam/waffledb`）または GitHub リポジトリ URL を指定する
- 指定がなければユーザーに確認する

## 手順

### 1. 対象リポジトリの確定

`$ARGUMENTS` から `owner/repo` を抽出する。URL 形式（`https://github.com/<owner>/<repo>`）も受け付ける。

### 2. ローカルクローンの確定

実装作業にはローカルクローンが必要。以下の順で探す:

1. カレントディレクトリが対象リポジトリのクローンか確認する（`git remote get-url origin`）
2. 設定ファイル `~/.vk-agents/config.json` の `workspace.search_paths` 以下を `find` で検索する
3. 見つからなければユーザーにローカルクローンのパスを確認する

以降の git 操作はすべてそのパスで行う。

### 3. 必要ラベルの確認・作成

```bash
gh label list --repo <owner/repo> --json name
```

不足しているラベルを作成する:

- **実装中**（存在しない場合）:
  ```bash
  gh label create "実装中" --repo <owner/repo> --color "fbca04" --description "実装作業中。"
  ```
- **実装済**（存在しない場合）:
  ```bash
  gh label create "実装済" --repo <owner/repo> --color "0e8a16" --description "実装完了・PR 作成済み。"
  ```

### 4. 「実装待」issue の一覧取得

```bash
gh issue list --repo <owner/repo> --label "実装待" --json number,title,body,url --limit 5
```

- 0 件の場合は「実装待の issue はありません」と報告して終了する
- 取得した issue を **番号の昇順** に並べ、最大 5 件を処理対象とする

### 5. 各 issue の処理（ループ・最大 5 件）

以下を 1 件ずつ繰り返す。5 件に達したら、残りを処理せず 6 へ進む。

#### 5-1. 「実装中」ラベルに切り替える（着手宣言）

```bash
gh issue edit <number> --repo <owner/repo> --remove-label "実装待" --add-label "実装中"
```

#### 5-2. ブランチを作成する

`develop` ブランチを元に、issue 番号から `no<number>` という名前のブランチを作成する:

```bash
cd <local_clone_path>
git fetch origin
git checkout develop
git pull origin develop
git checkout -b no<number>
```

#### 5-3. docs/ への実装ドキュメントを追記する

**ファイル名の決め方**:

1. `docs/` 内の既存ファイルを確認し、最大の連番を取得する:
   ```bash
   ls docs/ | grep -E '^[0-9]+_' | sort -n | tail -1
   ```
2. 最大番号 + 1 を新しい番号とする（例: 最大が `45_xxx.md` なら `46`）
3. ファイル名は `docs/<番号>_<issue内容を英語スネークケースで>.md` とする（例: `46_default_sort_order.md`）

**ドキュメントの内容**:

issue のタイトル・本文・設計セクション（あれば）をもとに、以下を含む実装ドキュメントを書く:

```markdown
# <issue タイトル>

## 概要
（何を実装したか）

## 実装内容
（変更したファイル・追加した機能の説明）

## 使い方
（ユーザー向けの操作説明。該当する場合）

## 技術的な補足
（実装の詳細・注意点。開発者向け）
```

#### 5-4. コードを実装する

issue の本文・設計セクション（あれば）を読み、実際のコード変更を行う。

実装の基本方針:
- issue 本文の「## 設計」セクションがあればそれに従う
- なければ issue のタイトル・説明から最小限の実装を判断する
- 既存コードのスタイル・命名規則・ファイル構成に合わせる
- WordPress プラグインとしての規約（`ABSPATH` チェック、nonce 検証、`$wpdb->prepare()` 等）を守る
- マイグレーションが必要な場合は `includes/waffledb_install.php` の既存パターンに従う

#### 5-5. 変更をコミットする

```bash
cd <local_clone_path>
git add -p   # 変更内容を確認しながら追加
git commit -m "[ 機能追加 ] <issue タイトルを体言止めで>"
```

コミットメッセージ形式は `[ 機能追加 ]` / `[ 改善 ]` / `[ 修正 ]` から内容に合うものを選ぶ。

#### 5-6. プッシュして PR を作成する

```bash
git push origin no<number>
```

PR 本文は以下の形式で作成する:

```bash
gh pr create \
  --repo <owner/repo> \
  --title "[ 機能追加 ] <issue タイトルを体言止めで>" \
  --base develop \
  --head no<number> \
  --body "$(cat <<'EOF'
## 概要
<実装内容の要約>

## 対応 issue
close #<number>

## 変更内容
- <変更点 1>
- <変更点 2>

## 確認手順
- [ ] <動作確認手順 1>
- [ ] <動作確認手順 2>
EOF
)"
```

#### 5-7. 「実装済」ラベルに切り替える

```bash
gh issue edit <number> --repo <owner/repo> --remove-label "実装中" --add-label "実装済"
```

#### 5-8. 進捗を報告する

```
✅ #<number> <title>
   ブランチ: no<number>
   docs: docs/<ファイル名>
   PR: <PR URL>
```

### 6. 結果の報告

処理完了後にまとめて報告する:

- 実装した issue の一覧（番号・タイトル・PR URL）
- 5 件に達して停止した場合はその旨と残り件数
- 実装の判断が難しかった点や、ユーザーへの確認事項（あれば）

## 注意事項

- **最大 5 件で必ず停止する**。5 件目の PR 作成後は次の issue を取得しない。
- ブランチ `no<number>` がすでに存在する場合はスキップしてユーザーに報告し、次の issue へ進む。
- コード実装の内容が issue の情報だけでは判断できない場合は、ユーザーに確認してから進む（推測で実装しない）。
- `develop` ブランチが存在しない場合はユーザーに確認する（`main` や `master` へのフォールバックは自動で行わない）。
- PR の base ブランチは常に `develop`。
- ラベル操作のエラーは処理を止めず、エラー内容を記録して続行する。
