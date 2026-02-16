# Change Detection Impact Analysis: helix

Analyzing last 10 commits...
Total workspace crates: 14

| Commit | Build | Test | Docs | Infra | Crates | Description |
|--------|-------|------|------|-------|--------|-------------|
| 74075bb | **OFF** | **OFF** | off | ON | 0/14 | Inject 'regex' into `diff.*.{xfuncn |
| 8cd8681 | **OFF** | **OFF** | ON | ON | 0/14 | adjust and extend the json-related  |
| 22b2e49 | ON | ON | off | ON | 0/14 | fix(docker-compose/bake): incorrect |
| 98911ed | ON | ON | off | ON | 10/14 | build(deps): bump the rust-dependen |
| 41d6b33 | ON | ON | off | ON | 5/14 | build(gix): enable `max-performance |
| 066dded | ON | ON | off | ON | 0/14 | Add pytest-language-server (#15269) |
| 86fcb5d | ON | ON | off | ON | 5/14 | build(deps): bump gix from 0.78.0 t |
| db5a48d | ON | ON | off | ON | 2/14 | build(deps): bump the rust-dependen |
| d12a48a | ON | ON | off | ON | 2/14 | build(deps): bump the rust-dependen |
| 81a3861 | ON | ON | off | ON | 0/14 | feat: add rail.toml for unification |

## Summary

| Metric | Value |
|--------|-------|
| Commits analyzed | 10 |
| Could skip build | 2 (20%) |
| Could skip tests | 2 (20%) |
| Targeted (not full run) | 2 (20%) |

**Interpretation:**
- Without cargo-rail: All 10 commits run full build+test
- With cargo-rail: 2 commits skip tests, 2 skip build
- Potential CI reduction: ~20% fewer test runs

---

*Generated on 2026-02-15 with cargo-rail v0.10.2*
