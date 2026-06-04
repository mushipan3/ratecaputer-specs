# Claude Desktop GitHub MCP セットアップ手順

**最終更新：2026-06-04**

---

## 概要

Claude Desktop に GitHub MCP サーバーを設定することで、claude.ai（スマホ・PC）では不可能な **GitHub への直接読み書き** が可能になる。

---

## 1. 設定ファイルの場所

Claude Desktop の MCP 設定ファイルは以下のパスにある：

```
%LOCALAPPDATA%\Packages\Claude_pzs8sxrjxfjjc\LocalCache\Roaming\Claude\claude_desktop_config.json
```

エクスプローラーのアドレスバーに上記をそのまま貼り付けるとアクセスできる。

> **注意**：`%APPDATA%\Claude\` ではなく、上記の Packages 配下のパスが正しい。

### 「設定を編集」ボタンから開く方法

1. Claude Desktop を起動する
2. 右上のメニュー（ハンバーガーアイコン）→「設定」を開く
3. 「開発者」タブ →「設定を編集」ボタンをクリック
4. `claude_desktop_config.json` がテキストエディタで開く

この方法が最も確実に正しいファイルを開ける。

---

## 2. 設定ファイルの正しいJSON構造

既存の `preferences` キーと **共存させる** 形で `mcpServers` を追加する。

```json
{
  "preferences": {
    // 既存の設定がある場合はそのまま残す
  },
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

> **注意**：`preferences` を削除しないこと。既存の設定が消えるとClaude Desktopの動作に影響する。

---

## 3. GitHub Personal Access Token の準備

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 「Generate new token (classic)」をクリック
3. 必要なスコープを選択：
   - `repo`（プライベートリポジトリへのアクセスに必要）
   - `read:org`（必要に応じて）
4. 生成されたトークン（`ghp_` で始まる文字列）をコピーして設定ファイルに貼り付ける

---

## 4. 設定反映手順

設定ファイルを保存しただけでは反映されない。**完全終了→再起動** が必要。

### 完全終了方法

通常の×ボタンではタスクトレイに常駐したままになる。  
以下のコマンドでプロセスを完全に終了させること：

```cmd
taskkill /F /IM claude.exe
```

PowerShell でも同様に実行できる。

### 再起動

`taskkill` 後、スタートメニューまたはデスクトップのショートカットから Claude Desktop を起動する。

---

## 5. 動作確認

Claude Desktop 起動後、チャットで以下を試す：

```
mushipan3/ratecaputer-specs リポジトリのルートにあるファイル一覧を教えてください
```

GitHub の情報が返ってくれば MCP 連携は成功。

---

## 6. トラブルシューティング

| 症状 | 原因 | 対処 |
|------|------|------|
| MCPが認識されない | 設定ファイルのパスが間違っている | 「設定を編集」ボタンから開いて確認 |
| JSONパースエラー | JSONの構文ミス（カンマ・括弧の過不足） | バリデーターで確認（例：jsonlint.com） |
| 認証エラー | トークンの権限不足またはExpired | トークンを再生成して貼り直す |
| 設定が反映されない | タスクトレイ常駐のまま | `taskkill /F /IM claude.exe` で完全終了後に再起動 |
| `preferences`が消えた | 上書き保存時に誤削除 | GitHubのコミット履歴から復元するか再設定 |
