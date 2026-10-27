# Standards

A standard here is a repeatable procedure: something we do more than once,
written so the next person — or the next agent session — runs it the same
way without re-deriving it. Each file is a playbook with commands to run and
a definition of done, not an essay. Plans in `docs/plans/` record one run of a
standard; the standard itself stays current.

| File | Use it when |
|---|---|
| [`cleanup-pass.md`](cleanup-pass.md) | A feature push has landed and the code needs to get smaller, shared and better tested across `pikol-backend` and `pikol-frontend`, without changing behaviour. |
