---
description: Test Maker (test-maker.app) で穴埋め・選択式・多答・別解ありの自動採点テストを作成・更新・一覧取得・内容取得する時に使う。ユーザーが「テストメーカーで問題作って」「穴埋めテスト作りたい」「〇〇のクイズを作って」「さっき作ったテストを直して」「問3だけ選択肢に変えて」などと依頼したとき、または `mcp__test-maker__create_examination` 等の MCP ツールを呼ぶ前に参照する。
---

# Test Maker Skill

[Test Maker](https://www.test-maker.app/) は穴埋め・選択式・多答問題を自動採点できるオンラインテスト作成ツール。このスキルは **MCP 経由でテストの作成・更新・一覧取得・内容取得** を行うときのガイド。

## 前提

- Test Maker アカウントでログイン済み（無料 / PRO どちらでも可）
- Test Maker MCP サーバーに接続済み

接続方法がまだの場合は [references/client-setup.md](references/client-setup.md) を参照してユーザーに案内する（Claude Code / Cursor / Claude Desktop / Codex CLI の設定手順を掲載）。

## 使える MCP ツール

| ツール | 用途 |
|---|---|
| `create_examination` | マークダウンから新しいテストを作成 |
| `update_examination` | 既存テストを上書き保存（差分編集） |
| `get_examination` | 既存テストをマークダウンで取得（差分編集の前段） |
| `list_my_examinations` | 自分のテスト一覧（枠情報・最新20件） |
| `list_examination_tags` | 公開テスト作成時に指定する `tagId` の一覧 |
| `get_account_status` | 現在のプラン・枠消費状況 |
| `get_test_maker_format_spec` | マークダウン記法の公式仕様（**セッション 1 回だけ** 取得してキャッシュ） |
| `get_feature_catalog` | MCP でできること / 編集ページでできることの一覧（ユーザーから機能を問われたら参照） |

## 典型ワークフロー

### 1. 新規作成（最頻出）

ユーザーが題材（教科書抜粋・PDF のテキスト・記事・手入力）を渡してきたら:

1. 題材からマークダウンで `<question correct="..." />` タグを埋め込んだ原稿を作成する
2. `create_examination({ title, markdown })` を呼ぶ（既定で **非公開** 作成）
3. 成功レスポンスに含まれる **回答ページ URL と編集ページ URL をユーザーに必ず提示**
4. デザインカスタマイズ・配信期間・回答者ログイン必須などを加えたい場合は **編集ページへ誘導**

マークダウン記法の詳細（`<question>` タグ全属性・誤答選択肢の作り方・完成例）は [references/format-spec.md](references/format-spec.md) を参照。

### 2. 差分編集（「問3だけ直して」「選択肢を1つ足して」など）

1. 対象の `examinationId` を特定する
   - 直前の `create_examination` / `update_examination` 応答に含まれる id を使う
   - 会話内に id が無ければ `list_my_examinations` を呼んで **新しい順の一覧から特定**（ユーザーに id を聞き返す前にまず呼ぶ）
2. `get_examination({ examinationId })` で **現在のマークダウンを取得**
3. ユーザーの指示に従って該当部分だけ編集
4. `update_examination({ examinationId, title, markdown })` で上書き保存

原文を取らずに全文書き直すと精度が落ちる。必ず `get_examination` → 編集 → `update_examination` の順で。

### 3. 一覧確認（「いくつ作った？」「さっきのどこ？」）

`list_my_examinations()` を呼ぶ。応答先頭に無料枠の消費状況（非公開 X / 3・公開 Y / 10）が含まれるので、ユーザーに残り枠も併せて案内する。

### 4. 機能確認（「〇〇できる？」「デザイン変えたい」）

`get_feature_catalog()` を呼ぶ。MCP 経由で今できること / 編集ページでのみ設定できること / 将来 MCP 対応検討中のことの一覧が返る。**MCP に無い機能は編集ページ URL に誘導** するのが基本。

## 重要な運用ルール

- **作成・更新後は回答ページ URL と編集ページ URL を必ずユーザーに提示**。省略しない
- **削除機能は MCP に無い**。テスト削除は Web UI（編集ページ or 管理画面）から行うようにユーザーに案内する
- 無料プランの枠は **非公開 3 件 / 公開 10 件**。上限到達時の対応:
  - 既存テストを `update_examination` で上書きする
  - PRO にアップグレードする（https://www.test-maker.app/m/payment）
  - 削除は MCP からはできないので、Web UI へ誘導
- **記述式問題（手動採点）は現時点で MCP 非対応**。必要な場合は編集ページから追加するよう案内
- `get_test_maker_format_spec` はセッション初回のみ呼ぶ（毎回呼ばない）

## トラブル時

- 認証切れ・401 エラー・枠制限・`tagId` エラー・round-trip の崩れなど: [references/troubleshooting.md](references/troubleshooting.md)

## マークダウン記法のミニマル例

```markdown
## 基礎英単語

りんごの英語は<question correct="apple" />である。

光合成が行われる細胞小器官は<question correct="葉緑体" choiceItems='["ミトコンドリア", "葉緑体", "リボソーム", "ゴルジ体"]' isMultipleChoice="false" />である。

日本の与党第一党は<question correct='["自民党", "自由民主党"]' hasAlternativeSolutions="true" />である。
```

詳細仕様（属性全リスト・数式・正誤判定・多答・品質ルール）は [references/format-spec.md](references/format-spec.md) を参照。
