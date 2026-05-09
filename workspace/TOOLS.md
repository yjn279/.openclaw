
# TOOLS.md - Local Notes

Skills define _how_ tools work. This file is for _your_ specifics — the stuff that's unique to your setup.

## What Goes Here

Things like:

- Camera names and locations
- SSH hosts and aliases
- Preferred voices for TTS
- Speaker/room names
- Device nicknames
- Anything environment-specific

## Examples

```markdown
### Cameras

- living-room → Main area, 180° wide angle
- front-door → Entrance, motion-triggered

### SSH

- home-server → 192.168.1.100, user: admin

### TTS

- Preferred voice: "Nova" (warm, slightly British)
- Default speaker: Kitchen HomePod
```

## Why Separate?

Skills are shared. Your setup is yours. Keeping them apart means you can update skills without losing your notes, and share skills without leaking your infrastructure.


Add whatever helps you do your job. This is your cheat sheet.

## Related

- [Agent workspace](/concepts/agent-workspace)

## Slack

ゆうじくんの Slack workspace の channel は `#XXX-YYY[-Z]` 形式で命名されている:
- **`XXX`** = 3 桁プロジェクト番号（`0XY` = monet 系、`1XY` = project 系）
- **`YYY[-Z]`** = カテゴリ prefix + project codename + 任意の派生 suffix

### 解釈ルール

| グループ | 構造 | repo codename の取り出し方 |
|---|---|---|
| **0XX 系** (monet umbrella) | `#0XX-monet[-suffix]` | `YYY` 直接 = `monet`（最終 `-suffix` は**派生扱い**で無視） |
| **1XX 系** (project umbrella) | `#1XX-project-<codename>[-suffix]` | `project-` の**次のセグメント** = `<codename>`（最終 `-suffix` は派生扱いで無視可） |

### 推測 cwd 一覧

| Channel | 推測 cwd |
|---|---|
| `#000-monet` | `~/Documents/monet` |
| `#090-monet-random` | `~/Documents/monet`（最終 `-random` 派生 → 無視） |
| `#100-project` | デフォルト workspace（meta channel、特定 repo なし） |
| `#110-project-brewia` | `~/Documents/brewia` |
| `#120-project-links` | `~/Documents/links` |
| `#130-project-vsaas` | `~/Documents/vsaas`（未作成。無ければ default workspace に fallback） |
| `#general` / `#random` / DM | デフォルト workspace `~/.openclaw/workspace` |

### 動作の前提

このマッピングは bot が channel 名を見て「**どの repo の話か脳内で意識する**」ためのヒント。

bot の tool calls の cwd は単一 workspace 固定なので、特定 repo を扱いたい時は user が絶対パスで明示する。

## `/trinity` メンションが来たときのルーティング

ユーザーが Slack で `<@モネ> /trinity ...` または `<@モネ> /trinity:run ...` のようにメンションしてきた場合、その文字列はモネ自身では実行できない。**Claude Code 用のスラッシュコマンド**であって、モネの `claude --print` モードでは単なるテキストにしかならず、ハングして 600s タイムアウトになる。

### 必ずこの手順で ACP に流す

1. まず `acp-router` スキルを読む（`mcp__openclaw__memory_get` ではなく、ACP runtime extension が提供している。`/acp` を使えと言わない、`subagents` runtime も使わない）。
2. `mcp__openclaw__sessions_spawn` を呼ぶ：`runtime: "acp", agentId: "claude", mode: "run", task: "...", cwd: "..."`
   - accepted で childSessionKey/runId が返る。ここで Trinity が実際に走る。
   - **`runTimeoutSeconds` を渡してはいけない**。実証データ：付けると ACP child が ~10s で `ACP_TURN_FAILED: Internal error` で死ぬ。原因不明だが付けない時は通る。
3. `task`/`cwd`：
   - `task`: ユーザー本文の `/trinity` を `/trinity:run` に正規化したもの。GitHub URL や `<owner>/<repo>#N` も task にそのまま含める。
   - `cwd`: ユーザーが触っているローカルリポジトリの絶対パス。codename → cwd の対応は **上の `## Slack` の「推測 cwd 一覧」**（channel `#1XX-project-<codename>` → `~/Documents/<codename>`）と同じ codename 体系。GitHub URL の `<owner>/<repo>` の `<repo>` 部分が codename と一致する想定。マッピングに無いリポを言われたら、まず `gh repo clone` を提案する（勝手に clone しない）。
4. accepted になったらユーザーに「Trinity に投げたよ〜」みたいな短い完了予告を返す。
5. ACP child の結果は子セッション通知で返ってくるので、モネはそれを要約してユーザーに返信する。

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

### スレッドフォローについて

`mode: "run"` は oneshot なので、ユーザーがスレッドで追加メッセージを送っても**同じ ACP child は再開できない**（毎回新規 oneshot）。継続作業が必要な場合は、ACP child から提示された判断材料（選択肢など）にユーザーが返事 → モネが新しい task として詳細化して再 spawn、という運用にする。これも上流バグが直れば解消する一時的な制約。
