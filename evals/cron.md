# Cron and Scheduler

Verify scheduled jobs run and route correctly.

## Steps

1. Add a temporary test job to `cron/config.yaml`:
   ```yaml
   - name: test
     cron: "* * * * *"
     prompt: "Say exactly: CRON_TEST_OK"
   ```
2. Start Logos and wait up to 60 seconds
3. Check the database for a message with conversationId containing "test" and text containing "CRON_TEST_OK"
4. Remove the test job from `cron/config.yaml`

## NO_REPLY behavior

1. The heartbeat job should fire every 30 minutes (or adjust to `* * * * *` temporarily)
2. When it fires with nothing to report, the agent should respond with `NO_REPLY`
3. Confirm no `NO_REPLY` string is stored in the database
