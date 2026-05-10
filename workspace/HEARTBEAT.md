# HEARTBEAT.md

- OpenClawの稼働状況を確認し、トラブルが発生している場合は解消してください。
- 下記の **Rate-limit Recovery** を毎回実行する（条件を満たさなければ即スキップ）。

## Rate-limit Recovery

Trinity / ACP の child run が rate-limit で `failed` のまま放置されているタスクを、デフォルトの Claude モデル（`anthropic/claude-opus-4-7`）が復旧したタイミングで元の Slack スレッドに再投入する。Ollama の状態は問わない（Claude が生きていなければ何もしない）。

### Steps

1. **Health check** — `openclaw infer model run --model anthropic/claude-opus-4-7 --prompt "ping" --json` を実行。返ってきた JSON が `"ok": true` でなければここで終了（Opus がまだ rate-limit / 障害中）。
2. **対象抽出** — `openclaw tasks list --status failed --json` を取り、以下の条件を **すべて** 満たすタスクだけを残す：
   - `error` に `"hit your limit"` を含む（rate-limit 起因）
   - `runtime` が `acp` または `subagent`（Trinity / ACP の child run）
   - `requesterSessionKey` が `agent:main:slack:` で始まる（Slack スレッド由来）
   - `task` が空でない
3. **再投入除外** — 状態ファイル `memory/heartbeat-rate-limit-resubmitted.json`（形式: `{"taskIds": ["..."]}`、無ければ `{"taskIds": []}` で作成）に既に記録されている `taskId` はスキップ。
4. **再投入** — 残った各タスクについて `sessions_send(sessionKey=<requesterSessionKey>, message=<task>)` で元の Slack スレッドに送信。送信に成功したら `taskId` を上記 JSON に追記して保存。
5. **通知** — 1件以上再投入したら、その件数だけ簡潔に報告（例: `2件のrate-limit failed taskを再投入したよ`）。0件なら `HEARTBEAT_OK`。

## Related

- [Heartbeat config](/gateway/config-agents)
