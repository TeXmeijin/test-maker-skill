# 各 AI クライアントの MCP セットアップ手順

Test Maker MCP サーバーに接続するための、主要な AI クライアント別の設定方法。

> 共通: 本番 MCP サーバーの URL は `https://api.test-maker.app/mcp`（OAuth 2.1 認可・Streamable HTTP トランスポート）。初回接続時にブラウザで Test Maker のログイン → 許可の画面が開く。

---

## Agent Skill の配置について（共通知識）

[Agent Skills 仕様](https://agentskills.io) は Anthropic 発祥の **SKILL.md ベースのオープン標準** で、2026 年時点では Cursor・Codex CLI・Gemini CLI・GitHub Copilot など 30 超のツールが採用している。多くのクライアントは以下の場所からスキルを自動検出する:

- プロジェクト単位: `.agents/skills/<name>/SKILL.md`
- ユーザー単位: `~/.agents/skills/<name>/SKILL.md`

加えて、各クライアント独自の場所（`.claude/skills/`, `.cursor/skills/`, `~/.codex/skills/` など）も読みに行く。クライアント横断では `gh skill install` が最も簡単（後述）。

> このリポジトリの skill 本体は `skills/test-maker/` 配下。Claude Code 以外のクライアントに手動で入れる場合は **`skills/test-maker/` ディレクトリをスキルディレクトリに配置** する（リポジトリルートではない）。

---

## Claude Code

### 方法1: CLI コマンド（推奨）

```bash
claude mcp add test-maker --transport http https://api.test-maker.app/mcp
```

追加後に `/mcp` で test-maker を選ぶと OAuth 認可が走る。

### 方法2: `.mcp.json` を手書き

プロジェクトルートの `.mcp.json` に追加:

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

### スキル（このリポジトリ）もインストールする場合

```bash
/plugin install github:TeXmeijin/test-maker-skill
```

インストール直後からスキルが有効になる（別途 enable は不要）。

---

## Cursor

### MCP サーバーを追加

1. 設定 → MCP Servers → `Edit mcp.json` を開く
2. 以下を追加:

```json
{
  "mcpServers": {
    "test-maker": {
      "url": "https://api.test-maker.app/mcp"
    }
  }
}
```

3. Cursor を再起動（または MCP 設定を reload）
4. 初回ツール呼び出し時にブラウザで OAuth 認可画面が開く

Cursor は設定ファイル形式が Claude Code とほぼ同じだが、`type` フィールドは省略可。

### スキルを追加（Cursor 2.4+）

Cursor は `.cursor/skills/`, `.agents/skills/`, `~/.cursor/skills/`, `~/.agents/skills/` を自動で読み込む（[公式ドキュメント](https://cursor.com/docs/skills)）。

**方法1: `gh skill` CLI（推奨・クロスエージェント）**

```bash
gh skill install TeXmeijin/test-maker-skill test-maker --agent cursor
```

**方法2: Cursor GUI**

1. Cursor Settings（`Cmd+Shift+J`）→ Rules → Project Rules → **Add Rule** → **Remote Rule (GitHub)**
2. URL に `https://github.com/TeXmeijin/test-maker-skill` を入力

**方法3: 手動 clone**

```bash
# ユーザー単位（推奨）
git clone https://github.com/TeXmeijin/test-maker-skill.git /tmp/tms && \
  mkdir -p ~/.cursor/skills && \
  cp -r /tmp/tms/skills/test-maker ~/.cursor/skills/test-maker

# または プロジェクト単位（チーム共有、git 管理）
git clone https://github.com/TeXmeijin/test-maker-skill.git /tmp/tms && \
  mkdir -p .cursor/skills && \
  cp -r /tmp/tms/skills/test-maker .cursor/skills/test-maker
```

設置後は新しい Cursor セッションから自動で検出される（再起動不要）。

---

## Claude Desktop

1. 設定 → Developer → Edit Config で `claude_desktop_config.json` を開く
2. 以下を追加:

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

3. Claude Desktop を完全終了 → 再起動
4. 新規チャットを開いて「テストメーカーで穴埋め作って」などと話しかけると初回 OAuth 認可が走る

> **注意**: Claude Desktop はバージョンによって HTTP 系 MCP のサポート状況が変わる。繋がらない場合は Claude Code / Cursor を使うか、[claude.ai/settings の Connectors](https://claude.ai/settings/connectors) から追加する手もある。

---

## Codex CLI

### MCP サーバーを追加

Codex CLI（OpenAI）の MCP 設定ファイル `~/.codex/config.toml`（ or プロジェクトの `.codex/config.toml`）に追加:

```toml
[mcp_servers.test-maker]
url = "https://api.test-maker.app/mcp"
```

接続方式は Streamable HTTP。Codex CLI 側が OAuth 2.1 対応のバージョンであれば、初回ツール呼び出し時にブラウザで認可フローが走る。

### スキルを追加

Codex CLI は `.agents/skills/`（repository scope）、`~/.agents/skills/`（user scope）、`/etc/codex/skills`（admin）を自動で読み込む（[公式ドキュメント](https://developers.openai.com/codex/skills)）。

**方法1: `gh skill` CLI（推奨）**

```bash
gh skill install TeXmeijin/test-maker-skill test-maker --agent codex
```

**方法2: 手動 clone**

```bash
# ユーザー単位
git clone https://github.com/TeXmeijin/test-maker-skill.git /tmp/tms && \
  mkdir -p ~/.agents/skills && \
  cp -r /tmp/tms/skills/test-maker ~/.agents/skills/test-maker
```

Codex は新しいスキルを自動検出する（progressive disclosure でメタデータだけ先に読み、使用時に SKILL.md 本体をロード）。

---

## その他の MCP クライアント

基本的に **Streamable HTTP + OAuth 2.1** に対応している MCP クライアントなら、以下の URL を設定すれば動く:

- URL: `https://api.test-maker.app/mcp`
- Transport: Streamable HTTP（POST ベース）
- Auth: OAuth 2.1（PKCE、Dynamic Client Registration 対応）
- OAuth Metadata: `https://api.test-maker.app/.well-known/oauth-protected-resource`

スキルも同様で、`.agents/skills/` または `~/.agents/skills/` を読み込むクライアントならこのリポジトリの `skills/test-maker/` をそこに置けばよい。横断的に管理するなら [`gh skill`](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/) が便利。

---

## 接続確認

設定後、以下の動作を確認:

1. 任意のテスト作成依頼（例「試しに穴埋め2問作って」）を AI に投げる
2. 初回は OAuth 認可のためブラウザが開く → Test Maker にログイン → 「許可」を押す
3. テストが作成され、応答に「回答ページ URL」「編集ページ URL」が表示される

認証や接続で詰まった場合は [troubleshooting.md](troubleshooting.md) を参照。
