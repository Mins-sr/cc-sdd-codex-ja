---
name: tp-spec-quick
description: インタラクティブモードまたは自動モードでのクイックな仕様の生成をします
---


# クイック仕様 ジェネレータ (tp-spec-quick)

<instructions>
## 重要事項（CRITICAL）: Automatic Mode (オートマティックモード)の実行ルール

**`$ARGUMENTS` に `--auto` フラグが含まれている場合、それはAUTOMATIC MODE(自動モード)であるという扱いです。**

Automatic Mode(自動モード) において:
- 全4つのフェーズを停止することなく連続したループで実行してください。
- 各フェーズの後に進行状況を表示してください。 (例: "Phase 1/4 complete: spec initialized")
- フェーズ2から4の「Next Step（次のステップ）」のメッセージは全て無視してください。 (それらは単独で使用するためのものです)
- フェーズ4のあと、終了する前に最終的なサニティレビューを実行してください。
- サニティレビューが完了した場合、またはエラーが発生した場合にのみ停止してください。

---

## コアタスク
4つの仕様フェーズを順次実行します。自動モードでは、すべてのフェーズを停止せずに実行します。インタラクティブモードでは、フェーズ間でユーザーに承認を求めます。

クイック生成の完了を主張する前に、生成された要件、設計、タスクに対して軽量なサニティレビューを 1 回実行します。マルチエージェントが利用できる場合は、フレッシュな（空の）サブエージェントを使用します。そうでない場合は、サニティレビューをインラインで実行します。

## 実行手順

### ステップ 1: 引数の解析と初期化

`$ARGUMENTS`の解析:
- `--auto` が含まれている場合: **Automatic Mode(自動モード)** (全4つのフェーズを実行する)
- 含まれていない場合: **Interactive Mode(インタラクティブモード)** (各フェーズでプロンプトを待つ)
- feature名とDescriptionの抽出 (含まれている場合は `--auto` フラグを文字列から取り除く)

例:
```
"User profile with avatar upload --auto" → mode=automatic, description="User profile with avatar upload"
"User profile feature" → mode=interactive, description="User profile feature"
```

モードのバナーを表示し、ステップ2に進みます。

### ステップ 2: フェーズループの実行

以下の4つのフェーズを順番に実行します:

---

#### フェーズ 1: 仕様の初期化 (Direct Implementation / 直接的な実装)

**コアラジック**:

1. **Brief(要約)の確認**:
   - `.exec-plans/specs/{feature-name}/brief.md` が存在する場合 (これは `$tp-discovery` によって作成されます)、それを読んでディスカバリーのコンテキスト（問題、アプローチ、範囲、制約など）を取得します。
   - `$ARGUMENTS` の代わりに、それをプロジェクトの説明として使用します。

2. **Feature(機能)名の生成**:
   - 説明（description）をケバブケース(kebab-case)に変換します。
   - 例: "User profile with avatar upload" → "user-profile-avatar-upload"
   - 名前は簡潔に維持すること（理想的には2〜4語）。

3. **ユニーク（唯一）であるかの確認**:
   - Glob を使用して `.exec-plans/specs/*/` をチェックします。
   - `brief.md` のみが存在する（`spec.json` がない）ディレクトリがある場合は、そのディレクトリを使用します（discovery が作成したため）。
   - そうでなく、機能名がすでに存在している場合は、`-2`, `-3` などを追加します。

4. **ディレクトリの作成**:
   - Bash環境を使用: `mkdir -p .exec-plans/specs/{feature-name}` (discoveryから既に存在している場合はスキップします)

5. **テンプレートからのファイルの初期化**:

   a. これらのテンプレートを読み込みます:
   ```
   - .exec-plans/settings/templates/specs/init.json
   - .exec-plans/settings/templates/specs/requirements-init.md
   ```

   b. プレースホルダーを置き換えます:
   ```
   {{FEATURE_NAME}} → feature-name
   {{TIMESTAMP}} → 現在の ISO 8601 タイムスタンプ (`date -u +"%Y-%m-%dT%H:%M:%SZ"` を使用します)
   {{PROJECT_DESCRIPTION}} → description
   ja → 言語コード (ユーザーの入力言語から検出します。デフォルトは `en` です)
   ```

   c. ファイルの書き込みに Write Tool を使用します:
   ```
   - .exec-plans/specs/{feature-name}/spec.json
   - .exec-plans/specs/{feature-name}/requirements.md
   ```

6. **進行状況の表示**: "Phase 1/4 complete: Spec initialized at .exec-plans/specs/{feature-name}/" ("フェーズ 1/4 完了: .exec-plans/specs/{feature-name}/ に仕様が初期化されました")

**自動モード**: フェーズ2に すぐに（IMMEDIATELY）進みます。

**インタラクティブモード**: 質問 "Continue to requirements generation? (yes/no)" ("要件の生成に進みますか？(yes/no)") でプロンプトを出します
- "no" の場合: 停止し、現在の状態を表示します。
- "yes" の場合: フェーズ2に進みます。

---

#### フェーズ 2: 要件の生成

`$tp-spec-requirements {feature-name}` を実行(Invoke)します。

完了を待ちます。この場合の「Next Step (次のステップ)」メッセージはすべて無視してください (これらは単独で使用するためのものです)。

**進行状況の表示**: "Phase 2/4 complete: Requirements generated" ("フェーズ 2/4 完了: 要件が生成されました")

**自動モード**: フェーズ3に すぐに (IMMEDIATELY) 進みます。

**インタラクティブモード**: 質問 "Continue to design generation? (yes/no)" ("設計の生成に進みますか？(yes/no)") でプロンプトを出します
- "no" の場合: 停止し、現在の状態を表示します。
- "yes" の場合: フェーズ3に進みます。

---

#### フェーズ 3: 設計の生成

`$tp-spec-design {feature-name} -y` を呼び出します。 `-y` フラグによって要件（Requirements）は自動承認されます。

完了を待ちます。「Next Step (次のステップ)」メッセージはすべて無視してください。

**進行状況の表示**: "Phase 3/4 complete: Design generated" ("フェーズ 3/4 完了: 設計が生成されました")

**自動モード**: フェーズ4に すぐに(IMMEDIATELY) 進みます。

**インタラクティブモード**: プロンプト "Continue to tasks generation? (yes/no)" ("タスクの生成に進みますか？(yes/no)")
- "no" の場合: 停止し、現在の状態を表示します。
- "yes" の場合: フェーズ4に進みます。

---

#### フェーズ 4: タスクの生成

`$tp-spec-tasks {feature-name} -y` を呼び出します。 `-y` フラグは、要件、設計、およびタスクを自動承認します。

完了を待ちます。

**進行状況の表示**: "Phase 4/4 complete: Tasks generated" ("フェーズ 4/4 完了: タスクが生成されました")

#### 最後の「健全性の確認(Sanity Review)」

フェーズ 4 の後、完了を宣言する前に軽量なサニティ(健全性)レビューを実行します。

- ディスクから直接 `requirements.md`、`design.md`、`tasks.md` をレビューします。 `brief.md` が存在する場合は、それをサポートするコンテキストとしてのみ使用します。
- マルチエージェントが利用できる場合は、フレッシュなレビューサブエージェントを使用することを推奨します。 ファイル パスとレビューの目的のみを渡します。 レビュー担当者は生成されたファイルを自ら読み取って精査を行う必要があります。
- レビューの焦点:
   - 要件、設計、タスクは首尾一貫したストーリー(全体のまとまりある文脈)を語っているか？
   - 必要な設計作業について、明らかな矛盾、不足している前提条件、またはタスクカバレッジの欠落はないか？
   - `_Depends:_`、`_Boundary:_` および `(P)` マーカーは、実装において妥当性があるか？
- レビューでタスク計画のローカルな（局所的な）問題のみが見つかった場合は、生成された `tasks.md` を 1 回修復また更新し、その後サニティレビューを再実行します。
- もしレビューによって現実の要件/設計でのギャップや矛盾が発見された場合は完了であるとは主張（宣言）せず、停止してフォローアップ処理を行ってください。

**これで、全4つのフェーズとサニティレビューが完了しました。**

最終完了サマリー(Final completion summary)を出力し（この下の Output Description の項目を参照）、終了します。

---

## 重要な制約事項

### エラー処理
- どの段階での失敗でもワークフローの進行は停止されます
- エラーの内容と「現在の進捗状況」が表示されます。
- そこから手動で再開する復帰コマンドをユーザに提案してください

</instructions>

## 出力の説明

### 実行モードのバナー表示

**Interactive Mode(インタラクティブモード) での表示**:
```
Quick Spec Generation (Interactive Mode) - クイック仕様生成 (インタラクティブモード)

このモードでは各フェーズごとに同意を求めるプロンプトが出されます。
Note: ギャップ分析(gap analysis)と設計の検証(design validation)ステップはスキップされます。
```

**Automatic Mode(自動モード) での表示**:
```
Quick Spec Generation (Automatic Mode) - クイック仕様生成 (自動モード)

このモードではすべてのフェーズがプロンプトなしで自動的に実行されます。
Note: オプションの検証ステップ(gap analysis, design review)とユーザー承認の確認はスキップされます。内部レビューのゲート機能そのものは引き続き動作・実行されています。
最終的なサニティレビューは引き続き実行されます。
```

### 中間進行状況の出力

各フェーズの後に、簡単な進行状況を表示します:
```
Spec initialized at .exec-plans/specs/{feature}/
Requirements generated → Continuing to design...
Design generated → Continuing to tasks...
```

### 最終的な完了サマリー (Final Completion Summary)

`spec.json` に指定されている言語（日本語）で結果を出力します:

```
クイック仕様生成（Quick Spec Generation）が完了しました！

## 生成されたファイル:
- .exec-plans/specs/{feature}/spec.json
- .exec-plans/specs/{feature}/requirements.md ({X} 件の要件)
- .exec-plans/specs/{feature}/design.md ({Y} コンポーネント, {Z} エンドポイント)
- .exec-plans/specs/{feature}/tasks.md ({N} 件のタスク)

当該クイック生成にてスキップされた確認ステップ:
- `$tp-validate-gap` - ギャップ分析 (他のコードとの統合チェック)
- `$tp-validate-design` - 設計レビュー (検証を伴った確認)

Sanity review (健全性の検証): [ PASSED (合格) | FOLLOW-UP REQUIRED (問題あり:フォローアップが必要) ]

## 次のステップ:
1. 生成された仕様書を確認してください (特に design.md)
2. (オプション) 検証を行なってください:
   - `$tp-validate-gap {feature}` - 既存のコードベースと統合されるかチェックします
   - `$tp-validate-design {feature}` - アークテクチャの品質を検証します
3. 実装を開始してください: `$tp-impl {feature}`

```

## 安全性とフォールバック

### エラーのシナリオ

**Template Missing (テンプレートの欠落)**:
- `.exec-plans/settings/templates/specs/` 内に存在するか確認する
- 足りない特定のファイルを報告(レポート)する
- エラーを出して終了する

**ディレクトリ作成の失敗**:
- 権限の確認
- 対象のパスと共にエラーを報告する
- エラーを出して終了する

**Phase Execution Failed / 各フェーズでの実行の失敗** (フェーズ2から4の間):
- ワークフローを停止する
- 現時点での進捗状況と完了したフェーズの一覧を表示する
- "`$tp-spec-{next-phase} {feature}` のコマンドから手動で再開してください" と提案する

**Sanity Review Failed / サニティレビューに不合格になった場合**:
- ワークフローを停止する。
- 矛盾の内容、前提条件となる項目の不足などのタスク計画(Task-plan)での問題を正確にレポート(報告)する。
- 指摘のあった状況に応じて、`$tp-spec-design {feature}` や `$tp-spec-tasks {feature}` を使ってフォローアップするか、もしくは手動（マニュアル）で修正するよう提案する。

**ユーザー操作によるキャンセル** (Interactive Mode限定):
- プログラム処理をごく自然に停止する
- 完了したフェーズを提示表示する
- 手動で再開できるようにサジェスト(提案)する
