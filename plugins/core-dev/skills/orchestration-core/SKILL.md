---
name: orchestration-core
description: 複数ステップ（3ステップ以上または複数ファイル）のタスクを委譲・受入判定する際に使用。サブエージェントへの委譲、モデル選択（Haiku/Sonnet/Opus/Fable/Codex）、委譲プロンプト作成、成果物の受入判定の前に読む。「実装して」「調査して」「リファクタして」「修正して」「計画して」「レビューして」等の作業依頼の開始時、「どのモデルに任せる」「委譲」「並列で」「コスト削減」「トークン節約」と言われた時に使用。単発の質問応答と、1〜2行の自明な修正だけの依頼では使用不可（後者はメインが直接 Edit する）。
---

# オーケストレーション実行手順

> 注（チーム配布版）: Codex（`codex-brief-review`・`/codex:review`・`codex:codex-rescue`）は任意であり、使えない場合は実装者と別の Claude モデルでレビューする（`core-dev:code-reviewer`）。

本スキル（SKILL.md と references/）がモデルルーティング・委譲の実行規律・エスカレーション・結果の検証の正である。メインセッションの役割（委譲と受入判定。直接行うのは読み取り専用の証跡確認と1〜2行の自明な修正）は `CLAUDE.md`（プロジェクトまたはユーザーのもの）が定める。メインのモデルによらず同じ手順で委譲する。

## 1. モデルルーティング（即決表）

該当する行があればそのモデルに割り当てる。該当しない場合だけ `references/routing.md`「Step 2: 判定基準」で決める。

| タスクの性質 | 割当 |
|---|---|
| 一括置換・整形・ログ収集・リンク/パス実在チェック・単純抽出・機械的な動作確認（テスト実行） | Haiku |
| 単一ファイル修正・定型テスト追加・通常のコードベース探索・資料の Markdown 変換 | Sonnet |
| Web 検索・軽量な調査 | Haiku / Sonnet + WebSearch |
| 複雑バグの根本原因調査・難解なデバッグ・アーキテクチャ横断リファクタ・特に複雑または重要な実装と調査・ライブラリ仕様の深掘り | Opus |
| 計画立案（実装計画・設計書・PRD のドラフト） | Opus（`core-dev:planner` / `Plan` / `model: opus`）。規模が大きい計画は Fable（`model: fable`）。付随する調査は Opus / Sonnet |
| コードレビュー・計画レビュー・レビュー観点のセカンドオピニオン | Codex（`/codex-brief-review` 経由。Luna・effort high）。routing.md の Codex ゲートを通す |
| 最重要タスクの検証とレビュー・重要な設計 | Fable（調査・実装には使わない） |
| 10件超の同型タスク | Workflow ツール（各段は Sonnet / Opus）。ユーザーが workflow の使用を明示した場合に使い、明示がなければ概算コストを示して確認する。異種のタスクは件数によらず Agent の並列起動 |
| 作業中にチームメイト同士の連絡が必要な作業 | Agent Teams（判定は routing.md、手順は `references/agent-teams.md`） |

- `model` を省略したときのモデルは、エージェント定義の `model` → 環境変数 `CLAUDE_CODE_SUBAGENT_MODEL` → メインのモデルの順で決まる。定義に model を持つエージェントはその model で動く（`core-dev:planner` は opus、`core-dev:code-reviewer` と `core-dev:security-reviewer` は sonnet）。メインと異なるモデルに割り当てるタスクでは `model`（haiku / sonnet / opus / fable）を明示する。`subagent_type: fork` はメインのモデルで動き `model` 指定が効かないため、メインのモデルに割り当ててよいタスクにだけ使う。
- 迷ったら `references/routing.md`「Step 2: 判定基準」で判定する。それでも迷えば一段安いモデルで試す。
- effort は指定しない（エージェント定義・環境変数・セッション設定に従う）。ユーザーが明示したレベルだけ渡す（routing.md「effort の指定」）。
- Codex はレビュー専用（`/codex-brief-review` 経由で Luna・effort high）。依頼前に routing.md の Codex ゲートを通す。
- Opus への委譲は1タスク単位に分け、委譲文にスコープを書く（routing.md「Opus コストガード」）。
- 同一タスクで2回失敗したら1段上のモデルへ引き上げる（routing.md「エスカレーション」）。

## 並列 fan-out（既定動作）

- 分解の既定粒度は「1機能＝1サブエージェント、1タスク＝1サブエージェント」。メインが独立単位に分解してから委譲する。
- 独立タスクは単一メッセージで複数エージェントを同時起動する。逐次は依存関係がある場合のみ。
- 実装の並列委譲では、メインが担当ファイルを分割し（ファイルパーティショニング）、同一ファイルは同時に1エージェントのみ。各委譲文に担当ファイルと触ってはならないファイルを明記する。

## 2. 委譲プロンプト必須6要素

委譲文は `references/delegation-templates.md` の雛形をコピーして書く。次の6要素を満たす:

1. **英語で書く**: 委譲プロンプト全文を英語で書き、返却も英語で受け取る（"Reply in English." を返却形式に含める）。サブエージェントと Agent Teams は英語を使うという運用規則に従う。ユーザーへの報告時にメインが日本語へ要約する。
2. **共通固定句**: 冒頭に `[NO RE-DELEGATION]` と `[PLAIN PROSE]` の2行を入れる（全文は delegation-templates.md「共通固定句」。チームメイト起動文は雛形7）。
3. **成果物定義**: 作成・変更するファイルと期待内容を具体的に。
4. **返却形式**: 結論・要約・file:line だけを返させ、生ログと全文貼付は不要と書く。進捗・推論・途中経過の出力も不要と書く（読むのはメインだけ）。
5. **検証可能性**: 変更ファイル一覧・`git diff --stat`（jj では `jj diff --stat`）・テスト結果の自己報告を要求。
6. **出所規律**: 自分がツール結果で裏取りできていない「事実」を委譲文に書かない（ハルシネーション連鎖防止）。

第一手段はメインによるファイル分担で書き込み先を分離すること。分担で衝突を回避できない場合のみ `isolation: "worktree"` を使う（通常のサブエージェント限定。`name` と混ぜない。agent-teams.md）。

## 3. 委譲後の受入判定と検証委譲

委譲完了ごとに `references/verification.md` の裏取り3点セットを回す。

- メインが直接行う（読み取り専用）: 差分確認（`git status` / `git diff --stat`。jj では `jj status` / `jj diff --stat`）と生成物の実在確認（`ls` / `wc -l` / 先頭数行の Read）。
- 委譲する: 動作確認（テスト実行・ビルド・実走）は検証用サブエージェントへ。レビューは実装と別のモデルへ（Claude 実装 → Codex）。
- Codex 返却の [CODEX-AUDIT] 判定は verification.md に従う。

## 4. トークン節約

委譲とパイプラインの設計時に `references/token-discipline.md` を適用する（重複呼び出しの排除、選択的実行、結論のみ返却）。

## 5. 呼び分け先

以下は既存資産を呼ぶ。本スキルで手順を再定義しない。

| 用途 | 呼び先 |
|---|---|
| Claude 実装分のレビュー | `/codex-brief-review`（フォールバックと手動レビューは verification.md「モデル分離レビュー」） |
| 補助レビュー（別サブエージェントによる差分レビュー。Codex レビューの代替にはしない） | `/code-review`（組み込み） |
| セキュリティ観点の差分レビュー | `/security-review`（組み込み）または `core-dev:security-reviewer` |

## Origin

user skills directory (orchestration-core)
