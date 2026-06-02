---
name: hanuman
description: Start or restart the Hanuman hub (Vue + Express app at localhost:3200). Handles crash recovery.
user-invocable: true
allowed-tools: Bash, Read
---

Start or restart the Hanuman hub (formerly claude-hub).

## Steps

1. Check if Hanuman is already running:
```bash
pgrep -f "projects/claude-hub/node_modules/.bin/concurrently" || true
```

2. If processes are found, kill the entire process tree:
```bash
pkill -f "projects/claude-hub" || true
sleep 1
```

3. Verify port 3200 is free:
```bash
lsof -ti :3200 | xargs kill -9 2>/dev/null || true
```

4. Start the hub in the background:
```bash
cd ~/projects/claude-hub && npm run dev
```
Run this with `run_in_background: true`.

5. Wait 2 seconds, then verify it's running:
```bash
sleep 2 && curl -s http://localhost:3200 > /dev/null && echo "Hanuman is running at http://localhost:3200" || echo "Failed to start"
```

6. Report the result to the user.

## Notes
- Project lives at `~/projects/claude-hub`
- Runs on port 3200 (Express server + Vite dev server via concurrently)
- If the user just says "restart hanuman" or "start hanuman", do the full cycle (kill + start)
