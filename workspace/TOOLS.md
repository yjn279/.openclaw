
# TOOLS.md

Skills define _how_ tools work. This file is for _your_ specifics — the stuff that's unique to your setup.

## Slack

Slack workspace の channel は `#XXX-YYY` 形式で命名されている:
- **`XXX`** : 3 桁プロジェクト番号。百の位はプレフィックス、十の位はプロジェクト名、一の位はサブチャンネルを表す。
- **`YYY`** : ハイフンで区切られた1〜3つの単語。一単語目はプレフィックス、二単語目はプロジェクト名、三単語めはサブチャンネルを表す。

## `/trinity` メンションが来たときのルーティング

ユーザーが Slack で `<@モネ> /trinity ...` または `<@モネ> /trinity:run ...` のようにメンションしてきた場合、その文字列はモネ自身では実行できない。**Claude Code 用のスラッシュコマンド**であって、モネの `claude --print` モードでは単なるテキストにしかならず、ハングして 600s タイムアウトになる。

### 必ずこの手順で ACP に流す

1. まず `acp-router` スキルを読む（`mcp__openclaw__memory_get` ではなく、ACP runtime extension が提供している。`/acp` を使えと言わない、`subagents` runtime も使わない）。
2. `mcp__openclaw__sessions_spawn` を呼ぶ：`runtime: "acp", agentId: "claude", mode: "run", task: "...", cwd: "..."`
   - accepted で childSessionKey/runId が返る。ここで Trinity が実際に走る。
   - **`runTimeoutSeconds` を渡してはいけない**。実証データ：付けると ACP child が ~10s で `ACP_TURN_FAILED: Internal error` で死ぬ。原因不明だが付けない時は通る。
3. `task`/`cwd`：
   - `task`: ユーザー本文の `/trinity` を `/trinity:run` に正規化したもの。GitHub URL や `<owner>/<repo>#N` も task にそのまま含める。
   - `cwd`: ユーザーが触っているローカルリポジトリの絶対パス。パスについてはチャンネルの説明文を参照する。
4. accepted になったらユーザーに「Trinityを開始したよ」のような短い完了予告を返す。
5. 仕様に関する質問が返却された場合、独断で判断せず必ずユーザーの判断を仰ぐ。
6. ACP child の結果は子セッション通知で返ってくるので、モネはそれを要約してユーザーに返信する。

### 例

ユーザー「`<@モネ> /trinity https://github.com/yjn279/links/issues/10`」

正しい行動：

```json
{
  "tool": "mcp__openclaw__sessions_spawn",
  "args": {
    "runtime": "acp", "agentId": "claude",
    "mode": "run",
    "task": "/trinity:run https://github.com/yjn279/links/issues/10",
    "cwd": "/Users/yuji/Documents/links"
  }
}
```

絶対にやってはいけないこと：

- `runTimeoutSeconds` を付ける（このフィールドが入ると ACP child が即死する。timeout は ACP runtime のデフォルトに任せる）
- `/trinity ...` の文字列をモネ自身がそのまま処理しようとする（ハングする）
- `subagents` runtime で動かそうとする（acp-router で禁止されている）
- ユーザーに「ターミナルで claude --rc を起動して」と言う（ACP runtime path が使えるなら不要）

