---
name: tkt-issue-design
description: "「設計待」ラベルが付いた GitHub issue を順番に処理し、設計内容を issue 本文に追記したうえで「設計待」を外し「設計確認待」を付ける。設計待ラベルがなくなるまで繰り返す。『設計待の issue を処理して』『設計を書いて』等の依頼時に使用"
---

# /tkt-issue-design スキル

「設計待」ラベルの付いた issue を順番に処理し、設計内容を本文末尾に追記してラベルを「設計確認待」に切り替える。これを「設計待」ラベルの issue がなくなるまで繰り返す。

## トリガー

以下の表現でこのスキルを使う:

- 「設計待の issue を処理して」
- 「設計を書いて」
- 「設計待をさばいて」
- 「tkt-issue-design」

## 引数

- `$ARGUMENTS` に対象リポジトリの URL または `owner/repo` 形式を指定する
- 指定がなければカレントディレクトリの git リポジトリ（`gh repo view` で解決できるもの）を使う
- それも確定できなければユーザーに確認する

## 手順

### 1. 対象リポジトリの確定

1. `$ARGUMENTS` から `owner/repo` を抽出する。URL 形式（`https://github.com/<owner>/<repo>`）も受け付ける。
2. 指定がなければ `gh repo view --json nameWithOwner` でカレントリポジトリを解決する。
3. 確定できなければユーザーに確認する。

### 2. 必要ラベルの存在確認

以下の 2 ラベルが存在するか確認し、なければ作成する:

```bash
gh label list --repo <owner/repo> --json name
```

- **設計中**（存在しない場合）:
  ```bash
  gh label create "設計中" --repo <owner/repo> --color "e4e669" --description "設計作業中。"
  ```
- **設計確認待**（存在しない場合）:
  ```bash
  gh label create "設計確認待" --repo <owner/repo> --color "0075ca" --description "設計を記入済み。レビュー待ち。"
  ```

### 3. 「設計待」issue の一覧取得

```bash
gh issue list --repo <owner/repo> --label "設計待" --json number,title,body,url --limit 100
```

件数が 0 件の場合は「設計待の issue はありません」と報告して終了する。

### 4. 各 issue の処理（ループ）

取得した issue を **番号の昇順** に 1 件ずつ処理する。

#### 4-1. 「設計中」ラベルに切り替える（着手宣言）

issue の処理を始める前に、まず「設計待」を外して「設計中」を付ける:

```bash
gh issue edit <number> --repo <owner/repo> --remove-label "設計待" --add-label "設計中"
```

#### 4-2. issue の詳細を取得する

```bash
gh issue view <number> --repo <owner/repo> --json number,title,body,labels,comments
```

#### 4-3. 設計内容を組み立てる

issue のタイトル・本文・既存コメント・ラベルを読み、以下の観点で設計を考える。

**必ず含める項目:**

- **概要**: この issue で実現すること（1〜2 文）
- **仕様**: ユーザーが見える動作・設定値・UI の具体的なふるまい
- **実装方針**: どのファイル・関数・テーブルを変更するか、どういうアプローチで実現するか

**内容に応じて含める項目（不要なら省略可）:**

- **データ構造・DB設計**: 新しいカラム・テーブル・オプション値の形式
- **UI/UX設計**: 画面の配置・ラベル・バリデーションなど
- **注意事項・制約**: 後方互換性・パフォーマンス・セキュリティなど既知のリスク

フォーマット（本文末尾に追記するマークダウン）:

```markdown

---

## 設計

### 概要
（ここに記載）

### 仕様
（ここに記載）

### 実装方針
（ここに記載）

### データ構造（省略可）
（ここに記載）

### UI/UX設計（省略可）
（ここに記載）

### 注意事項（省略可）
（ここに記載）
```

#### 4-4. issue 本文を更新する（設計を追記）

現在の本文を取得し、末尾に設計セクションを追記した新しい本文を一時ファイルに書き出して編集する:

```bash
# 本文を一時ファイルに書き出す
gh issue view <number> --repo <owner/repo> --json body --jq '.body' > /tmp/issue_body.md

# 設計セクションを末尾に追記
cat >> /tmp/issue_body.md << 'EOF'

---

## 設計
（設計内容）
EOF

# issue 本文を更新
gh issue edit <number> --repo <owner/repo> --body-file /tmp/issue_body.md
```

**重要**: 既存の本文を消さないよう、必ず追記（append）する。

#### 4-5. ラベルを切り替える（設計完了）

```bash
gh issue edit <number> --repo <owner/repo> --remove-label "設計中" --add-label "設計確認待"
```

#### 4-6. 進捗を報告する

```
✅ #<number> <title> — 設計を追記し、ラベルを「設計確認待」に変更しました
```

### 5. 次の issue へ

4 の処理を「設計待」ラベルの全 issue に繰り返す。全件処理が終わったら 6 へ。

### 6. 結果の報告

処理完了後に以下をまとめて報告する:

- 処理した issue の一覧（番号・タイトル・URL）
- 設計を書く際に判断が難しかった点（あれば）
- ユーザーへの確認事項・次のアクション提案（あれば）

## 注意事項

- issue 本文の既存内容は**絶対に削除しない**。追記のみ行う。
- 設計はそのリポジトリの技術スタック・コーディングルールに沿って書く。WordPress プラグインであれば PHP/WordPress の慣習に準じる。
- 「設計中」「設計確認待」ラベルが存在しない場合は事前に作成する（2 の手順）。
- ラベル操作のエラー（ラベル未存在など）は処理を止めず、エラー内容を記録して次の issue に進む。
- 全件処理が終わったあとに改めて `gh issue list --label "設計待"` で件数が 0 になっているか確認する。残っていれば原因を調べてユーザーに報告する。
