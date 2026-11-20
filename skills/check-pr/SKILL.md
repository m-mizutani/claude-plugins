---
name: check-pr
description: "現在のブランチのPRのレビューコメントとCIステータスを確認し、問題があれば修正する"
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# PRレビューコメント・CI確認・対応スキル

現在のブランチに紐づくPRのレビューコメントとCIの実行結果を確認し、問題があれば修正を行う。

## Part A: CIステータスの確認と修正

### Step A1: CIステータスの取得

以下のコマンドでPRに紐づくCI (GitHub Actions等) のステータスを確認する。

```bash
gh pr checks
```

### Step A2: 失敗したCIの調査

失敗しているチェックがある場合、以下のコマンドで詳細ログを取得する。

```bash
# 失敗したrunのIDを取得
gh run list --branch $(git branch --show-current) --status failure --limit 5 --json databaseId,name,conclusion

# 失敗したrunの詳細ログを確認
gh run view <run_id> --log-failed
```

### Step A3: CI失敗の修正

ログから原因を特定し、修正を行う。修正後、CLAUDE.mdに記載されたチェックコマンド（vet, lint, test等）をローカルで実行して修正が正しいことを確認する。

> 失敗しているCIチェックは、自分の変更が原因かどうかに関わらず全て修正する。このPRをマージ可能な状態にすることが目的なので、「自分のミスではない」「既存の問題」「flakyテスト」「mainでも失敗している」は修正しない理由にならない。
>
> - 原因がブランチの変更と無関係に見えても、必ず原因を調査し、このブランチ上で修正してマージ可能な状態にする
> - 環境問題・インフラ問題でローカル修正が不可能な場合のみ、その根拠を明示してユーザーに判断を仰ぐ。安易に「対象外」と判断しない
> - 「失敗しているCIが1つでも残っている状態」をPR完了とみなさない

### Step A4: CI結果のサマリー

CIチェックごとの結果（成功・失敗・修正内容）を控え、Part C のサマリーに記載する。全て成功している場合は Part B に進む。

## Part B: レビューコメントの確認と修正

### Step B1: コメント一覧の取得

以下のコマンドを実行して、PRの全レビュースレッドを取得する。

```bash
OWNER=$(gh repo view --json owner --jq '.owner.login')
REPO=$(gh repo view --json name --jq '.name')
PR_NUMBER=$(gh pr view --json number --jq '.number')

gh api graphql -f query='
  query {
    repository(owner: "'"$OWNER"'", name: "'"$REPO"'") {
      pullRequest(number: '"$PR_NUMBER"') {
        title
        url
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            path
            line
            startLine
            comments(first: 50) {
              nodes {
                body
                author { login }
                createdAt
                url
              }
            }
          }
        }
      }
    }
  }
' \
  --jq '{
    pr_number: '"$PR_NUMBER"',
    pr_title: .data.repository.pullRequest.title,
    pr_url: .data.repository.pullRequest.url,
    threads: [.data.repository.pullRequest.reviewThreads.nodes[] | {
      thread_id: .id,
      is_resolved: .isResolved,
      path: .path,
      line: .line,
      start_line: .startLine,
      comments: [.comments.nodes[] | {
        body: .body,
        author: (.author.login // "unknown"),
        created_at: .createdAt,
        url: .url
      }]
    }]
  }'
```

### Step B2: 対象スレッドの特定とTodoリスト作成

JSONの `threads` 配列から `is_resolved: false` のスレッドのみを抽出し、ファイルパスと行番号を付けて一覧にする。この一覧を上から1件ずつ処理する。

### Step B3: 各コメントの精査

各unresolvedスレッドについて、以下の手順で処理する:

1. スレッド内の**全コメントの全文**を注意深く読む
2. 必要に応じて、指摘対象のファイルをReadツールで読み込み、該当箇所のコンテキストを理解する
3. コメントの詳細が不足している場合は、以下のコマンドで追加情報を取得する:
   ```bash
   gh api graphql \
     -f id='<thread_id>' \
     -f query='
       query($id: ID!) {
         node(id: $id) {
           ... on PullRequestReviewThread {
             id
             isResolved
             path
             line
             startLine
             diffSide
             comments(first: 100) {
               nodes {
                 body
                 author { login }
                 createdAt
                 url
               }
             }
           }
         }
       }
     ' \
     --jq '{
       thread_id: .data.node.id,
       is_resolved: .data.node.isResolved,
       path: .data.node.path,
       line: .data.node.line,
       start_line: .data.node.startLine,
       diff_side: .data.node.diffSide,
       comments: [.data.node.comments.nodes[] | {
         body: .body,
         author: (.author.login // "unknown"),
         created_at: .createdAt,
         url: .url
       }]
     }'
   ```

### Step B4: 妥当性の評価

以下の観点でコメントの妥当性を判断する:

- **正確性**: コードのバグや誤りを指摘しているか
- **セキュリティ**: セキュリティリスクの改善を提案しているか
- **可読性・保守性**: コードの品質向上に寄与するか
- **プロジェクト規約**: プロジェクトの慣例・規約に沿った指摘か
- **実現可能性**: 提案された修正が実現可能で副作用がないか

### Step B5: 修正作業

妥当と判断した場合:
1. コメントで指摘されたファイルパスと行番号を参照し、該当ファイルをReadツールで開く
2. EditまたはWriteツールで修正を適用する
3. CLAUDE.mdに記載されたチェックコマンド（vet, lint, test等）を実行し、修正が既存コードに影響しないことを確認する

妥当でないと判断した場合は、コードを修正せず、Step B6 に進む。

### Step B6: Conversationのresolve

対応方針が確定したスレッドは、修正した場合も対応不要と判断した場合も、GitHub 上で解決済みにする。スレッドの解決は他の人も見る状態を変える操作なので、実行前に、解決済みにするスレッドとその影響を示して承認を得る。今回の依頼でスレッドの解決まで明示的に許可されている場合は、その承認を再び求めない。

スレッドは1件ずつ解決し、まとめて解決しない。対応不要と判断して解決したスレッドは、判断の理由を Part C のサマリーに書く。サマリーに理由がないと、ユーザーがその判断を確認できないためである。

以下のコマンドでconversationをresolveする:

```bash
gh api graphql \
  -f threadId='<thread_id>' \
  -f query='
    mutation($threadId: ID!) {
      resolveReviewThread(input: { threadId: $threadId }) {
        thread {
          id
          isResolved
        }
      }
    }
  ' \
  --jq '.data.resolveReviewThread.thread.isResolved'
```

`<thread_id>` はStep B1で取得したスレッドの `thread_id` フィールドの値を使用する。結果が `true` であればresolve成功。

GraphQL クエリは、このスキルに書かれた複数行の形のまま実行する。1行に圧縮するとパースエラーになることがある。GraphQL API の呼び出しで TLS エラーが出た場合は、一時的なエラーのことが多いので、同じコマンドを再試行する。

### Step B7: 次のコメントへ

一覧の次のunresolvedスレッドに対して、Step B3から繰り返す。

## Part C: 完了サマリー

全ての確認・修正が完了したら、以下の形式でサマリーを出力する:

```markdown
## PR確認結果

### CIステータス
| チェック名 | ステータス | 対応 |
|-----------|----------|------|
| build | pass | - |
| lint | fail | 修正済み |

### レビューコメント
| スレッド | ファイル | 対応 | 理由 |
|---------|---------|------|------|
| #1 | path/to/file:42 | 修正済み・resolved | [修正理由] |
| #2 | path/to/file:88 | 対応不要・resolved | [対応不要と判断した根拠] |
```

**対応不要と判断してresolveしたスレッドの報告は、以下を満たすこと。**

- 表の「対応」列に `対応不要・resolved` と記載し、resolve済みであることを明示する
- 「理由」列には、指摘の内容と、対応不要と判断した具体的な根拠（該当コードの実際の挙動、プロジェクトの規約、指摘の前提が成立しないこと等）を書く。「妥当でないと判断」「不要」のみの記述は認めない
- 判断根拠には、参照した `file:line` やコマンド出力など、ユーザーがサマリーだけを読んで判断の妥当性を再検証できる情報を含める
- 対応不要としたスレッドが1件以上ある場合は、表のあとにそのスレッドのコメントURLを列挙し、ユーザーが元コメントを直接確認できるようにする

ユーザーが判断を差し戻した場合は、該当スレッドをunresolveして修正に入る（`unresolveReviewThread` mutation を使う）。
