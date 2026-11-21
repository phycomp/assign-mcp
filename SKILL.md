---
name: assign-mcp
description: Use when assigning work to agents via OpenMausBot MCP — creating role-based bots, grouping them into channels, sending kickoff messages, and tracking task progress. Triggers on "assign task to bot", "OpenMausBot", "create channel", "kickoff team", "assign MCP task", or when deploying AI teammates for parallel work.
---

# Assign MCP (OpenMausBot Task Assignment)

Assign and track work across OpenMausBot agents (bots and channels) via MCP. Covers bot/channel discovery, task creation, kickoff messaging, progress tracking, and interruption.

## When to Use

- Deploying AI teammates for parallel work
- Assigning a task to a specific bot or a channel of bots
- Kicking off a team coordination session
- Checking progress of assigned tasks
- Interrupting or switching tasks

## Prerequisites

- OpenMausBot MCP server connected (`openmausbot` in MCP config)
- At least one bot or channel exists

## Core Workflow

### 1. Discover available agents

```
list_bots()     → bots with id, name, title, section, activeTaskId, tasks[]
list_channels() → channels with id, name, memberIds, activeTaskId, tasks[]
```

Always discover before assigning. Note each bot's `activeTaskId` and `tasks[]`.

### 2. Create a task

**For a single bot:**
```
create_task(target_type="bot", target_id=<bot_id>, title=<title>)
```

**For a channel (team):**
```
create_task(target_type="channel", target_id=<channel_id>, title=<title>)
```

- Returns `taskId` — save it for subsequent sends.
- The task becomes active immediately.

### 3. Send kickoff message

**To a bot:**
```
send_bot_message(bot_id=<id>, task_id=<taskId>, text=<instruction>)
```

**To a channel:**
```
send_channel_message(channel_id=<id>, task_id=<taskId>, text=<instruction>, mentions=[<bot_ids>])
```

- Use `@name` in text to direct specific bots.
- Use `mentions` array to disambiguate shared names.
- `send_id` (optional, 16-80 chars) for idempotency — resending with same id returns first receipt.

### 4. Track progress

```
get_bot_messages(bot_id, task_id, limit)     → recent messages
get_channel_messages(channel_id, task_id, limit) → recent messages
list_bots() / list_channels()                 → activeTaskId, busy, activity
```

Check `busy`, `activity`, `waitingForTeammates` to know if work is in progress.

### 5. Control tasks

- **Interrupt**: `interrupt_conversation(target_type, target_id, task_id)` — stop current turn without killing the task.
- **Switch task**: `switch_task(target_type, target_id, task_id)` — change active task.
- **Rename**: `rename_task(target_type, target_id, task_id, title)` — update task title.
- **Search**: `search_messages(query, task_id, limit)` — find specific content in transcripts.

## Gotchas

- **Task creation auto-activates** — the new task becomes the bot's active task. If the bot was busy on another task, switch back with `switch_task` when done.
- **Channel default responder** — check `defaultResponder` (kind: everyone/mentions/member). With `mentions`, only @-mentioned bots respond. **Important**: use specific bot names (e.g., `@CEO`, `@MarketSales`), not `@all` — `@all` may not trigger responses. The `mentions` array is for **disambiguation** (when multiple bots share a name), not for specifying recipients.
- **Busy channels can't switch** — `switch_task` on a busy channel will fail. Wait or interrupt first.
- **Idempotent sends** — use `send_id` to avoid duplicate messages on retry.
- **Model selection** — bots have their own model (instanceId/model). Check before assigning complex tasks; some models are weaker.
- **Channel bulletin** — the bulletin field sets shared instructions for all members. Update it when team context changes.
- **Messages may trigger external tools** — bots can use external tools (web search, file access). This is expected but monitor for unexpected actions.
- **External tool limitation** — bots may fail when trying to use desktop/browser tools if the OpenMausBot desktop app is not running. If a bot reports "Only the OpenMausBot app on this computer can let a bot use it", either open the desktop app or rephrase the task to avoid browser-dependent operations (e.g., ask for text-only research).
- **No response from channel** — if no bot responds after sending to a channel, check: (1) `defaultResponder` kind, (2) whether @mentions use correct bot names, (3) whether bots are busy. Try sending to individual bots instead of the channel.

## Progress Tracking

### Polling pattern

After sending a kickoff message, poll for results:

```
# Check bot status
list_bots() / list_channels()  → busy, activity, waitingForTeammates

# Read messages
get_bot_messages(bot_id, task_id, limit=20)
get_channel_messages(channel_id, task_id, limit=40)

# Check if done
activity == "idle" and busy == false → likely complete
```

### Timeout handling

- If a bot shows `busy=true` for > 5 minutes, consider interrupting: `interrupt_conversation(target_type, target_id, task_id)`
- If no response after 3 minutes, re-send with a simpler task or switch to a different bot
- For channel tasks, poll each member individually if the channel shows no activity

### Completion criteria

A task is complete when:
- Bot's `activity == "idle"` and `busy == false`
- At least one bot message (role="bot") exists after the kickoff
- The message contains substantive content (not just an error)

## Example: Assign 3 parallel tasks

```
# 1. Create tasks
t1 = create_task(target_type="bot", target_id=<mausbot_id>, title="Research A")
t2 = create_task(target_type="bot", target_id=<devbot_id>, title="Build B")
t3 = create_task(target_type="channel", target_id=<channel_id>, title="Team C")

# 2. Send kickoff messages
send_bot_message(bot_id=<mausbot_id>, task_id=t1.taskId, text="Research task description...")
send_bot_message(bot_id=<devbot_id>, task_id=t2.taskId, text="Build task description...")
send_channel_message(channel_id=<channel_id>, task_id=t3.taskId, text="@CEO @MarketSales @PR @VP Team task description...", mentions=[...])

# 3. Track
get_bot_messages(bot_id=<mausbot_id>, task_id=t1.taskId, limit=10)
get_channel_messages(channel_id=<channel_id>, task_id=t3.taskId, limit=20)

# 4. Interrupt if needed
interrupt_conversation(target_type="bot", target_id=<bot_id>, task_id=<taskId>)
```

## Anti-Patterns

- ❌ Don't send a message without creating a task first (use default task only for simple queries).
- ❌ Don't forget to include `task_id` in send — otherwise it goes to the bot's currently active task, which may not be the one you just created.
- ❌ Don't assume all bots in a channel will respond — check `defaultResponder` and use `mentions`.
- ❌ Don't send duplicate messages without `send_id` on retry.
- ❌ Don't use `@all` in channel messages — use specific bot names (`@CEO`, `@MarketSales`). `@all` may not trigger responses.
- ❌ Don't assign browser-dependent tasks to bots without verifying the OpenMausBot desktop app is running.
- ❌ Don't ignore `busy`/`activity` state — poll before sending follow-up messages.

## References
- OpenMausBot MCP tools: list_bots, list_channels, create_task, send_bot_message, send_channel_message, get_bot_messages, get_channel_messages, interrupt_conversation, switch_task, rename_task, search_messages
