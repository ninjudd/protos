# Startup

Verify the process starts and stops cleanly.

## Steps

1. Ensure `ANTHROPIC_API_KEY` is set in `.env`
2. Run `./logos start`
3. Confirm the wrapper reports the PID and the log shows "running"
4. Run `./logos status` — should report running
5. Run `./logos stop` — should stop cleanly
6. Run `./logos status` — should report not running
