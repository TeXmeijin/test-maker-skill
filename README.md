# Test Maker MCP Skill

**AI エージェントから [Test Maker](https://www.test-maker.app/) で穴埋め・選択式テストを作る**ための MCP サーバーと Claude Code Skill。

「教科書のこのページから穴埋め問題を10問作って」「さっき作ったテストの問3だけ選択式に変えて」と **普段使っている AI にそのまま喋るだけ** で、Test Maker にテストが保存されます。

---

## これは何？

- **Test Maker**: 穴埋め・選択・多答・別解ありの自動採点テストを作成・配信できる Web サービス（https://www.test-maker.app/）
- **MCP サーバー**: Claude Code / Cursor / Claude Desktop / Codex CLI などから Test Maker にテストを作成・更新できる公式 MCP エンドポイント
- **このリポジトリ**: Claude Code 向けのスキル（プロンプト資産）を配布します。MCP サーバー本体は Test Maker 側で運用しており、URL (`https://api.test-maker.app/mcp`) に繋ぐだけで使えます

## できること（MCP 経由）

| 機能 | ツール |
|---|---|
| マークダウンから新規テスト作成 | `create_examination` |
| 既存テストを差分編集で上書き | `update_examination` |
| 既存テストの内容をマークダウンで取得 | `get_examination` |
| 自分のテスト一覧 + 枠状況 | `list_my_examinations` |
| 公開テスト用のタグ一覧 | `list_examination_tags` |
| プラン・枠状況の確認 | `get_account_status` |
| マークダウン記法の公式仕様 | `get_test_maker_format_spec` |
| 機能カタログ（何ができる／編集ページへの案内） | `get_feature_catalog` |

MCP で直接設定できない項目（デザインカスタマイズ、配信期間、回答者ログイン必須、画像アップロード、記述式問題など）は、**各テストの編集ページから Web UI で設定** できます（今後 MCP 対応を検討）。

---

## インストール

### Claude Code

MCP サーバーとスキルの両方を追加:

```bash
# MCP サーバーを追加
claude mcp add test-maker --transport http https://api.test-maker.app/mcp

# このスキルをインストール
/plugin install github:TeXmeijin/test-maker-skill
```

初回にテストを作ろうとするとブラウザで OAuth 認可画面が開くので、Test Maker にログイン → 「許可」を押してください。以降は AI に `「〇〇のテスト作って」` と話すだけで動きます。

### Cursor

MCP サーバー — `.cursor/mcp.json` または設定画面の MCP Servers に追加:

```json
{
  "mcpServers": {
    "test-maker": {
      "url": "https://api.test-maker.app/mcp"
    }
  }
}
```

スキル（任意）:

```bash
# CLI でインストール（GitHub CLI 2026-04 以降）
gh skill install TeXmeijin/test-maker-skill test-maker --agent cursor

# または GUI: Settings → Rules → Add Rule → Remote Rule (GitHub)
```

### Claude Desktop

`claude_desktop_config.json` に追加:

```json
{
  "mcpServers": {
    "test-maker": {
      "type": "http",
      "url": "https://api.test-maker.app/mcp"
    }
  }
}
```

### Codex CLI

MCP サーバー — `~/.codex/config.toml` に追加:

```toml
[mcp_servers.test-maker]
url = "https://api.test-maker.app/mcp"
```

スキル（任意）:

```bash
gh skill install TeXmeijin/test-maker-skill test-maker --agent codex
```

### その他の MCP クライアント

- URL: `https://api.test-maker.app/mcp`
- Transport: Streamable HTTP
- Auth: OAuth 2.1（PKCE + Dynamic Client Registration 対応）

**スキルについて**: このリポジトリのスキルは [Agent Skills](https://agentskills.io) 仕様（SKILL.md ベース）に準拠。Cursor / Codex CLI / Gemini CLI / Copilot など 30 超のクライアントが `.agents/skills/` または `~/.agents/skills/` を読むので、`skills/test-maker/` ディレクトリをそこに置けば動く。クロスエージェントで一括管理するなら [`gh skill`](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/) が便利（`--agent claude-code|cursor|codex|copilot|gemini-cli|antigravity` で対象指定）。

---

## 使い方（例）

AI クライアントに以下のようなメッセージを投げると、MCP 経由でテストが作成されます。

```
日本史の江戸時代について、穴埋め問題を5問作って
```

```
生物の細胞小器官についての選択問題を作成したい。
ミトコンドリア・葉緑体・リボソーム・ゴルジ体を使った4択を3問。
```

```
さっき作った「江戸時代」テストの問3を選択式に変えて。選択肢は歴代将軍から4つ。
```

```
この PDF の内容から穴埋め問題を10問、別解ありにして作って。
```

作成が成功すると、AI が **回答ページ URL** と **編集ページ URL** を返します:

- **回答ページ**: このリンクを配布すれば誰でも回答できる
- **編集ページ**: Web UI でテストを開く。ここから追加の設定（デザイン・配信期間・ログイン必須など）が可能

---

## 料金・枠

| 項目 | 無料 | PRO |
|---|---|---|
| 非公開テスト作成数 | 3 件まで | 無制限 |
| 公開テスト作成数 | 10 件まで | 無制限 |
| 1 テストあたりの問題数 | 30 問まで | 無制限 |
| 解答履歴保存数 | 10 件まで | 300 件 |

枠に達したテストは MCP からは削除できません。**削除は Test Maker の Web UI から** 行ってください。

PRO のアップグレード: https://www.test-maker.app/m/payment

---

## FAQ

### スキルのインストールは必須？

必須ではないですが、インストールすると AI が **Test Maker 用の最適な呼び出し順序・マークダウン記法・運用ルール** を理解した状態で動くので、成功率が上がります。

スキル無しでも MCP サーバー単体で繋がります（AI がツール description を読んで適宜呼んでくれる）。

### 対応する AI クライアントは？

Streamable HTTP + OAuth 2.1 対応の MCP クライアントすべて。

動作確認済み: Claude Code（Anthropic）/ Cursor / Claude Desktop / Codex CLI。

### 記述式問題は作れる？

現時点では MCP 非対応です。編集ページ（Web UI）から追加してください。将来の MCP 対応を検討中。

### オープンソース？

**MCP サーバー本体**: Test Maker のソースコードはオープンソースではありません。エンドポイントのみ公開しています。

**このスキル (test-maker-skill)**: MIT ライセンスで公開。フォーク・改変自由。プロンプト改善の Pull Request 歓迎。

---

## リンク

- Test Maker 本体: https://www.test-maker.app/
- 課金ページ: https://www.test-maker.app/m/payment
- MCP サーバー URL: `https://api.test-maker.app/mcp`
- OAuth メタデータ: https://api.test-maker.app/.well-known/oauth-protected-resource
- このリポジトリ: https://github.com/TeXmeijin/test-maker-skill

## ライセンス

MIT — このリポジトリのみ。Test Maker 本体は別ライセンス。
