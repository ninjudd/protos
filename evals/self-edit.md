# Self-Edit and Restart

Verify the restart safety mechanism works.

## Type-check gate

1. Manually introduce a type error in a source file (e.g., add `const x: number = "bad";`)
2. Run `./logos restart`
3. Confirm the type check fails and the restart is aborted
4. Confirm the old process is still running (if it was running)
5. Revert the type error
