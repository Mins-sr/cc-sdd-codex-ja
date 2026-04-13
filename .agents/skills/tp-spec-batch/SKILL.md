---
name: tp-spec-batch
description: roadmap.md にあるすべての機能についての完全な仕様（要件、設計、タスク）を、依存関係の順序（wave）ごとに並行サブエージェントディスパッチを使用して作成します。
---


# 仕様バッチ作成 (tp-spec-batch)

<background_information>
- **成功基準**:
  - すべての機能について完全な仕様ファイル (spec.json, requirements.md, design.md, tasks.md) が揃っている
  - 依存関係の順序が尊重されている（下流の仕様の前に上流の仕様が完了する）
  - 独立した機能はサブエージェントのディスパッチを介して並行して処理される
  - 仕様間の整合性が検証されている（データモデル、インターフェース、命名規則など）
  - `## Specs (dependency order)` の解析を壊すことなく、ロードマップ上の混在したコンテキストを理解できている
  - コントローラーのコンテキストは軽量に保たれている（サブエージェントが重い作業を行う）
</background_information>

<instructions>

## ステップ 1: ロードマップの読み込みとバリデーション

1. `.exec-plans/steering/roadmap.md` を読み込む
2. `## Specs (dependency order)` セクションを解析し、以下を抽出する:
   - 機能名
   - 1行の説明
   - 各機能の依存関係
   - 完了ステータス（`[x]` = 完了, `[ ]` = 未完了）
3. 存在する場合は、コンテキスト把握のために以下も読み込む:
   - `## Existing Spec Updates`
   - `## Direct Implementation Candidates`
   これらを依存関係ウェーブ(波)での実行には含めないこと。これらは順序付けと整合性レビューのための認識専用の入力である。
4. `## Specs (dependency order)` の中の未完了の各機能について、`.exec-plans/specs/<feature>/brief.md` が存在することを検証する。
5. もし brief.md が一つでも欠けている場合は、停止して「次の brief.md が不足しています: [リスト]。まずは `$tp-discovery` を実行して brief を作成してください。」と報告する。

## ステップ 2: 依存関係の波 (Waves) を構築する

依存関係に基づいて、未完了の機能を波(ウェーブ)にグループ化する:

- **Wave 1**: 依存関係がない機能（またはすべての依存関係がすでに `[x]` として完了している機能）
- **Wave 2**: 依存関係がすべて Wave 1 にある、またはすでに完了している機能
- **Wave N**: 依存関係がすべて以前の Wave にある、またはすでに完了している機能

実行計画を表示する:
```
Spec Batch Plan:
  Wave 1 (parallel): app-foundation
  Wave 2 (parallel): block-editor, page-management
  Wave 3 (parallel): sidebar-navigation, database-views
  Wave 4 (parallel): cli-integration
  Total: 6 specs across 4 waves
```

ロードマップに `## Existing Spec Updates` または `## Direct Implementation Candidates` が含まれている場合は、ユーザーが分解の全体像を見ることができるように、それらをバッチ対象外の項目として別途言及すること。

## ステップ 3: Wave の実行

各 Wave について、Wave 内のすべての機能を **並列サブエージェント** としてディスパッチする。

**Wave 内の各機能について**、次のタスクを持つサブエージェントを生成する:

```
機能 "{feature-name}" の完全な仕様を作成してください。

1. 機能のコンテキストを把握するため、.exec-plans/specs/{feature-name}/brief.md にある概要（brief）を読み込む
2. プロジェクトのコンテキストを把握するため、.exec-plans/steering/roadmap.md にあるロードマップを読み込む
3. フル仕様のパイプラインを実行する。各フェーズにおいて、完全な指示（テンプレート、ルール、レビューゲート）について、対応するスキルの SKILL.md を読み込むこと:
   a. 初期化: .agents/skills/tp-spec-init/SKILL.md を読み込んでから、spec.json と requirements.md を作成する
   b. 要件生成: .agents/skills/tp-spec-requirements/SKILL.md を読み込んでから、その手順に従う
   c. 設計生成: .agents/skills/tp-spec-design/SKILL.md を読み込んでから、その手順に従う
   d. タスク生成: .agents/skills/tp-spec-tasks/SKILL.md を読み込んでから、その手順に従う
4. spec.json 内のすべての承認（approvals）を true に設定する（自動承認モード、-y フラグと同じ）
5. ファイルのリストとタスク数と共に完了を報告する
```

マルチエージェントが利用できない場合は、Wave内の機能を順次実行する。

**Wave内のすべてのサブエージェントが完了した後**:
1. 各機能に spec.json, requirements.md, design.md, tasks.md があることを検証する
2. もし失敗した機能があれば、エラーを報告し、成功した機能だけで続行する
3. Wave の完了を表示する: "Wave N complete: [features]. Files verified."
4. 次の Wave に進む

## ステップ 4: 仕様横断的なレビュー (Cross-Spec Review)

すべての Wave が完了した後、仕様全体の整合性をレビューするために **単一のサブエージェント** を生成する。利用可能であれば `spec-reviewer` カスタムエージェント（`.codex/agents/spec-reviewer.toml` で `model = "gpt-5.4"` および `model_reasoning_effort = "high"` に設定されている）を使用する。これは最も価値の高い品質ゲートである。個別の仕様レベルのレビューゲートでは捉えきれない問題を見つけ出す。

**サブエージェントのタスク**:

生成されたすべての仕様を読み、プロジェクト全体の整合性を確認する:
- `.exec-plans/specs/*/design.md` (最重要: インターフェース、データモデル、アーキテクチャを含む)
- `.exec-plans/specs/*/requirements.md` (範囲と受入基準について)
- `.exec-plans/specs/*/tasks.md` (境界のアノテーションのみ -- `_Boundary:_` 行を読み、タスクの説明はスキップする)
- `.exec-plans/steering/roadmap.md`

読み込みの優先順位: design.md ファイル (インターフェース、データモデル、アーキテクチャを含む) を優先する。requirements.md では、セクションの見出しと受入基準に焦点を当てる。tasks.md では、`_Boundary:_` アノテーションに焦点を当てる。

確認事項:
1. **データモデルの一貫性**: 同じエンティティが仕様間で一貫して定義されているか (フィールド名、型、関係性)
2. **インターフェースの整合性**: 仕様Aが出力するものを仕様Bが消費する場合、契約 (コントラクト) は完全に一致しているか？
3. **機能の重複の排除**: 複数の仕様で指定されている機能はないか？
4. **依存関係の網羅性**: すべての design.md が正しい上流の仕様を参照しているか？ロードマップにない暗黙の依存関係はないか？
5. **命名規則**: コンポーネント名、ファイルパス、APIルート、テーブル名は仕様書全体で一貫しているか？
6. **共有インフラストラクチャ**: 共有される関心事 (認証、エラー処理、ロギング) が1つの仕様内で処理され、正しく参照されているか？
7. **タスク境界の整合**: タスクの `_Boundary:_` アノテーションによってコードベースは綺麗に分割されているか？ 複数の仕様によってクレーム(要求)されているファイルはないか？
8. **ロードマップ境界の連続性**: ロードマップに `Existing Spec Updates` または `Direct Implementation Candidates` が含まれている場合、生成された新しい仕様は、誤ってその作業を吸収していないか？
9. **アーキテクチャ境界の完全性**: 新しい仕様は、責任の継ぎ目をきれいに保ち、共有オーナーシップを避け、依存関係の方向を首尾一貫させ、下流への影響を捉えるのに十分な再検証トリガーを含んでいるか？
10. **変更しやすい分解**: 一緒にすべきでなく分割すべき複数の独立した継ぎ目(seams)を、単一の仕様内で吸収していないか？

出力: 一貫している(CONSISTENT)領域 ＋ 問題点(ISSUES)（どの仕様か、何が不整合なのか、提案される修正方法）。

**レビューのサブエージェントが戻った後**:
- **Critial(致命的) / Important(重要) な問題が発見された場合**: 提案された修正を適用するために、影響を受ける仕様ごとに修正サブエージェントをディスパッチする。問題が実際には分解の問題（例えば、境界の重複や、一つの仕様が複数の独立した継ぎ目を抱え込んでいるなど）である場合は、局所的にその場しのぎで対処するのではなく、停止してロードマップ/ディスカバリーに戻る。修正後、再度仕様横断的レビューを実行する（修正ラウンドは最大3回まで）。
- **軽微な(Minor)問題のみ**: ユーザーが認識できるように報告し、ステップ 5 に進む。
- **問題なし**: ステップ 5 に進む。

## ステップ 5: 完了確定

1. すべての仕様が存在することを確認するために `.exec-plans/specs/*/tasks.md` をスキャンする
2. 完了した各仕様について、spec.json を読んでフェーズと承認状況を確認する
3. roadmap.md を更新する: 完了した仕様に `[x]` をマークする
4. roadmap.md に `Existing Spec Updates` または `Direct Implementation Candidates` が含まれている場合は、すでに他で明示的に完了している場合を除き、手を触れずに残りのフォローアップ項目として言及する

最終サマリーを表示する:
```
Spec Batch Complete:
  ✓ app-foundation: X requirements, Y design components, Z tasks
  ✓ block-editor: ...
  ✓ page-management: ...
  ...
  Total: N specs created, M tasks generated
  Cross-spec review: PASSED / N issues found (M fixed)
  Existing spec updates pending: <count or none>
  Direct implementation candidates pending: <count or none>

次に、生成された仕様を確認し、$tp-impl <feature> で実装を開始してください。
```

</instructions>

## 重要な制約
- **コントローラーは軽量なままである**: メインコンテキストでは roadmap.md の読み取りと brief.md の存在確認のみを行う。仕様の生成はすべてサブエージェントで行われる。
- **Wave の順序は厳密である**: 前の Wave のすべての機能が完了するまで、絶対に次の Wave を開始しない。
- **Wave 内での並行処理**: マルチエージェントが利用できる場合は、同じ Wave 内のすべての機能が並行してディスパッチされるべきである。
- **部分的な Wave の禁止**: Wave 内の機能が失敗した場合でも、報告する前にその Wave の残りの機能は完了させること。
- **完了した仕様はスキップする**: roadmap.md で `[x]` となっている機能、または既存の tasks.md がある機能はスキップされる。
- **`## Specs (dependency order)` はバッチ実行のための権威として残る**: 他のロードマップのセクションはコンテキストであり、Wave の入力ではない。

## 安全性とフォールバック

**サブエージェントの失敗**:
- エラーを記録し、失敗した機能をスキップする
- Wave 内の残りの機能をそのまま続行する
- サマリーで失敗した機能を報告する
- 提案する:「失敗した機能については、手動で `$tp-spec-quick <feature> --auto` を実行してください。」

**循環依存**:
- 依存関係グラフに循環がある場合は、循環を報告して停止する
- 提案する:「roadmap.md の依存関係の順序を修正してください」

**ロードマップが見つからない**:
- 停止して報告する:「roadmap.md が見つかりません。まずは `$tp-discovery` を実行してください。」

**すべての仕様がすでに完了している**:
- 報告する:「roadmap.md のすべての仕様はすでに完了しています。何もする必要はありません。」
