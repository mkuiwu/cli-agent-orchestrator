# Provider Combo Example

This example defines a mixed-provider team:

- `kimi_supervisor` -> `kimi_cli`
- `codex_developer` -> `codex`
- `kimi_reviewer` -> `kimi_cli`

Additional combo:

- `codex_supervisor` -> `codex`
- `codex_developer` -> `codex`
- `kimi_developer` -> `kimi_cli`
- `kimi_reviewer` -> `kimi_cli`

## Install

```bash
cao install examples/provider-combo/kimi_supervisor.md
cao install examples/provider-combo/codex_developer.md
cao install examples/provider-combo/kimi_reviewer.md
cao install examples/provider-combo/codex_supervisor.md
cao install examples/provider-combo/kimi_developer.md
```

## Launch

```bash
cao launch --agents kimi_supervisor --provider kimi_cli

# Or use Codex as the supervisor
cao launch --agents codex_supervisor --provider codex
```

## Why this works

CAO uses the supervisor's provider as the default for spawned workers. A worker
profile can override that default with its own `provider` field. In this setup:

- the supervisor itself runs on Kimi
- `codex_developer` overrides the inherited provider and launches on Codex
- `kimi_reviewer` explicitly stays on Kimi

If you remove the `provider` field from a worker profile, that worker will
inherit the supervisor's provider instead.
