# Send and Receive

Verify the basic message loop works via the HTTP channel.

## Steps

1. Start Logos
2. Send a message via the HTTP channel:
   ```
   curl -X POST http://localhost:3000 -H "Content-Type: application/json" -d '{"text": "Hi, what is your name?"}'
   ```
3. Confirm the response contains text from the agent
4. Check the database — there should be one `user` message and one `assistant` message

## Memory round-trip

1. Send: `{"text": "Remember that my favorite color is blue"}`
2. Confirm the agent acknowledges
3. Send: `{"text": "What is my favorite color?"}`
4. Confirm the agent recalls blue
5. Check that a file exists in `memories/` with today's date
