# Autonomy and Safety

## Execution Model

| Feature | Behavior |
|---------|----------|
| Automatic task assignment | Router classifies and assigns |
| Automatic resource selection | Smallest capable resource picked |
| Automatic retries and fallback | Retry on failure, fallback to next resource |
| Automatic escalation | Escalate to stronger model if result insufficient |
| Approval for risky actions | User approval required for destructive ops |
| Isolated accounts and browser profiles | Per-account isolation enforced |
| Task checkpoints and recovery | State saved periodically |
| Verification before complete | Verification Engine checks output |

## Risky Actions Requiring Approval

- File deletion or overwrite outside scratch/
- Database writes or schema changes
- Secret/credential access
- Network calls to external services
- Code that will be committed or deployed

## Checkpoint Format

```json
{
  "task_id": "uuid",
  "phase": "execution",
  "resource": "nvidia-nim-kimi",
  "input": "...",
  "output_so_far": "...",
  "artifacts": ["path/to/file1", "path/to/file2"],
  "errors": [],
  "timestamp": "2026-11-05T14:00:00Z"
}
```

Checkpoints are written to disk and also recorded in the Hermes session.

## Verification Engine

Before marking a task complete:
1. Check output against task requirements
2. Run tests/linters if code was produced
3. Flag mistakes or incomplete work
4. Trigger rework (back to Router) if needed
