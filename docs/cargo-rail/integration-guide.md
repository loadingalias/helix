# cargo-rail Integration Guide for Helix

This guide explains how to integrate `cargo-rail` change detection into Helix's development workflow.

## Why cargo-rail?

Helix's CI runs on every PR with no path filtering. cargo-rail adds intelligent change detection that can:

1. **Skip entire test matrices for docs-only PRs** - Your 4-OS test matrix is expensive
2. **Detect theme/query-only changes** - Runtime assets that don't need full rebuild
3. **Target affected crates** - Test only 2-4 crates instead of all 14
4. **Provide local/CI parity** - `cargo rail plan --merge-base` shows exactly what CI will do

## Measured Impact (10 Recent Commits)

| Change Type | Build | Test | Crates | CI Savings |
|-------------|-------|------|--------|------------|
| **docs-only** | OFF | OFF | 0/14 | **Skip 4-OS test matrix** |
| **query-only** | OFF | OFF | 0/14 | Skip build/test |
| code change | on | on | 2-4/14 | Targeted testing |
| deps bump | on | on | 2/14 | Limited scope |

### Specific Examples

```
c7f4fe5 (docs): "special" is used in pickers
  Files: book/src/themes.md
  Result: Build OFF, Test OFF, Docs ON
  → Full test matrix SKIPPED

9b82876 (query): highlight "TIP" as hint
  Files: runtime/queries/comment/highlights.scm
  Result: Build OFF, Test OFF, custom:queries ON
  → Full rebuild SKIPPED

57c3668 (fix): regression in force write
  Files: helix-view/src/*.rs, tests/*.rs
  Result: Build ON, Test ON, 4 crates affected
  → Test only helix-view + dependents (not all 14)
```

## Configuration

The included `.config/rail.toml` is configured for Helix with **workflow-specific profiles**:

```toml
[change-detection]
infrastructure = [
  ".github/**",
  "flake.nix",
  "flake.lock",
  "rust-toolchain.toml",
  "languages.toml",  # Grammar cache key
]
confidence_profile = "balanced"
bot_pr_confidence_profile = "strict"

# Custom surfaces for runtime assets
[change-detection.custom]
themes = ["runtime/themes/**"]
queries = ["runtime/queries/**"]
book = ["book/**"]
xtask = ["xtask/**"]

# Map workflow jobs to profiles
[run.workflow]
build = "ci"
check = "msrv"
test = "test"
lints = "lint"
docs = "docs"

# Profile definitions
[run.profile.ci]
surfaces = ["build", "test"]
merge_base = true

[run.profile.msrv]
surfaces = ["build"]
merge_base = true

[run.profile.test]
surfaces = ["test"]
merge_base = true

[run.profile.lint]
surfaces = ["build"]
merge_base = true

[run.profile.docs]
surfaces = ["docs", "custom:themes", "custom:queries"]
merge_base = true
```

### Key Configuration Points

| Setting | Value | What It Does |
|---------|-------|--------------|
| `themes` surface | runtime/themes/** | Triggers `custom:themes` in plan output |
| `queries` surface | runtime/queries/** | Triggers `custom:queries` in plan output |
| `docs` profile | `["docs", "custom:themes", "custom:queries"]` | Runs when any docs-related surface activates |
| `[run.workflow]` | Job → profile mapping | `cargo rail run --workflow docs` uses docs profile |

## Integration Options

Helix already has `cargo xtask` for task automation. The recommended approach is to use `cargo rail plan` for change detection and let xtask handle execution.

### Recommended: `cargo rail plan` + xtask

cargo-rail handles change detection. xtask handles execution. Clean separation.

**Local development:**

```bash
# Install
cargo install cargo-rail

# See what changed and which surfaces are enabled
cargo rail plan --merge-base --explain

# Get plan as JSON for scripting
PLAN=$(cargo rail plan --merge-base -f json)

# Check if test surface is enabled
if echo "$PLAN" | jq -e '.surfaces.test.enabled' > /dev/null; then
  cargo xtask test
fi

# Check custom surfaces
THEMES=$(echo "$PLAN" | jq -r '.surfaces["custom:themes"].enabled')
if [ "$THEMES" = "true" ]; then
  cargo xtask theme-check
fi
```

**Why this approach?**

1. **cargo-rail stays focused**: Change detection only - which crates changed, which surfaces matter
2. **xtask stays authoritative**: Your existing task definitions don't change
3. **Composable**: CI consumes plan output, xtask runs the actual commands
4. **No lock-in**: If you stop using cargo-rail, xtask still works

### Alternative: `cargo rail run` (simpler, less flexible)

If you prefer a single command, `cargo rail run` wraps plan + execution:

```bash
cargo rail run --merge-base --surface test
cargo rail run --workflow docs --dry-run --print-cmd
```

This uses cargo-rail's built-in executors (`cargo test`, `cargo check`, etc.) rather than xtask.

### Full CI Integration with GitHub Action

Modify `build.yml` to use cargo-rail-action. **The action automatically renders a step summary** showing what changed and why in the GitHub UI.

```yaml
name: Build
on:
  pull_request:
  push:
    branches: [master]
  merge_group:
  schedule:
    - cron: "00 01 * * *"

env:
  MSRV: "1.87"
  GRAMMAR_CACHE_VERSION: ""

jobs:
  # Planning job - runs cargo-rail and outputs surface decisions
  plan:
    runs-on: ubuntu-latest
    outputs:
      build: ${{ steps.rail.outputs.build }}
      test: ${{ steps.rail.outputs.test }}
      docs: ${{ steps.rail.outputs.docs }}
      infra: ${{ steps.rail.outputs.infra }}
      themes: ${{ steps.custom.outputs.themes }}
      queries: ${{ steps.custom.outputs.queries }}
      base-ref: ${{ steps.rail.outputs.base-ref }}
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0  # Required for change detection

      # cargo-rail-action installs cargo-rail, runs plan, and renders step summary
      - uses: loadingalias/cargo-rail-action@v3
        id: rail
        with:
          mode: full           # Include custom surfaces
          args: '--explain'    # Show WHY in step summary

      # Extract custom surfaces from JSON output
      - name: Extract custom surfaces
        id: custom
        run: |
          THEMES=$(echo '${{ steps.rail.outputs.custom-surfaces }}' | jq -r '.["custom:themes"] // "false"')
          QUERIES=$(echo '${{ steps.rail.outputs.custom-surfaces }}' | jq -r '.["custom:queries"] // "false"')
          echo "themes=$THEMES" >> "$GITHUB_OUTPUT"
          echo "queries=$QUERIES" >> "$GITHUB_OUTPUT"

  # Check (MSRV) - runs when build surface is active
  check:
    name: Check (msrv)
    needs: plan
    if: needs.plan.outputs.build == 'true' || needs.plan.outputs.infra == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ env.MSRV }}
      - uses: Swatinem/rust-cache@v2
        with:
          shared-key: "build"
      - name: Cache tree-sitter grammars
        uses: actions/cache@v5
        with:
          path: runtime/grammars
          key: ${{ runner.os }}-${{ runner.arch }}-stable-v${{ env.GRAMMAR_CACHE_VERSION }}-tree-sitter-grammars-${{ hashFiles('languages.toml') }}
      - run: cargo check

  # Test Suite - runs when test surface is active
  test:
    name: Test Suite
    needs: plan
    if: needs.plan.outputs.test == 'true' || needs.plan.outputs.infra == 'true'
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest, ubuntu-24.04-arm]
    env:
      RAIL_SINCE: ${{ needs.plan.outputs.base-ref }}
    steps:
      - uses: actions/checkout@v6
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ env.MSRV }}
      - uses: Swatinem/rust-cache@v2
        with:
          shared-key: "build"
      - name: Cache tree-sitter grammars
        uses: actions/cache@v5
        with:
          path: runtime/grammars
          key: ${{ runner.os }}-${{ runner.arch }}-stable-v${{ env.GRAMMAR_CACHE_VERSION }}-tree-sitter-grammars-${{ hashFiles('languages.toml') }}
      # Use RAIL_SINCE for consistent change detection
      - run: cargo test --workspace
      - run: cargo integration-test

  # Lints - runs when build surface is active
  lints:
    name: Lints
    needs: plan
    if: needs.plan.outputs.build == 'true' || needs.plan.outputs.infra == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ env.MSRV }}
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
        with:
          shared-key: "build"
      - run: cargo fmt --all --check
      - run: cargo clippy --workspace --all-targets -- -D warnings
      - run: cargo doc --no-deps --workspace --document-private-items
        env:
          RUSTDOCFLAGS: -D warnings

  # Docs validation - runs when docs, themes, or queries surfaces are active
  docs:
    name: Docs
    needs: plan
    if: |
      needs.plan.outputs.docs == 'true' ||
      needs.plan.outputs.themes == 'true' ||
      needs.plan.outputs.queries == 'true' ||
      needs.plan.outputs.infra == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ env.MSRV }}
      - uses: Swatinem/rust-cache@v2
        with:
          shared-key: "build"
      - name: Cache tree-sitter grammars
        uses: actions/cache@v5
        with:
          path: runtime/grammars
          key: ${{ runner.os }}-${{ runner.arch }}-stable-v${{ env.GRAMMAR_CACHE_VERSION }}-tree-sitter-grammars-${{ hashFiles('languages.toml') }}

      # Only validate queries if query files changed
      - name: Validate queries
        if: needs.plan.outputs.queries == 'true' || needs.plan.outputs.build == 'true'
        run: cargo xtask query-check

      # Only validate themes if theme files changed
      - name: Validate themes
        if: needs.plan.outputs.themes == 'true' || needs.plan.outputs.build == 'true'
        run: cargo xtask theme-check

      - run: cargo xtask docgen
      - name: Check uncommitted documentation changes
        run: |
          git diff
          git diff-files --quiet \
            || (echo "Run 'cargo xtask docgen', commit the changes and push again" \
            && exit 1)

  # Upload decision receipts for debugging (always runs after jobs complete)
  upload-receipts:
    name: Upload Decision Receipts
    needs: [plan, check, test, lints, docs]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Summary
        run: |
          echo "## CI Summary" >> "$GITHUB_STEP_SUMMARY"
          echo "" >> "$GITHUB_STEP_SUMMARY"
          echo "| Surface | Status |" >> "$GITHUB_STEP_SUMMARY"
          echo "|---------|--------|" >> "$GITHUB_STEP_SUMMARY"
          echo "| Build | ${{ needs.plan.outputs.build }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Test | ${{ needs.plan.outputs.test }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Docs | ${{ needs.plan.outputs.docs }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Themes | ${{ needs.plan.outputs.themes }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Queries | ${{ needs.plan.outputs.queries }} |" >> "$GITHUB_STEP_SUMMARY"
          echo "| Infra | ${{ needs.plan.outputs.infra }} |" >> "$GITHUB_STEP_SUMMARY"
```

### What You'll See in GitHub UI

When the action runs with `args: '--explain'`, the **step summary** shows:

```
## Change Detection Summary

plan

changed files: 2
  runtime/themes/omicron_dark.toml [custom:themes] -> unowned
  runtime/themes/omicron_light.toml [custom:themes] -> unowned

direct crates: 0
transitive crates: 0

surfaces:
  build: off
  test: off
  docs: off
  custom:themes: on (2 reason(s))
  custom:queries: off
  infra: off

Skipping: check, test, lints
Running: docs (theme validation only)
```

This gives **full visibility** into why jobs ran or were skipped.

## Custom Surfaces

The config defines custom surfaces for Helix's runtime assets:

| Surface | Pattern | When Active | What Runs |
|---------|---------|-------------|-----------|
| `custom:themes` | runtime/themes/** | Theme file changes | theme-check |
| `custom:queries` | runtime/queries/** | Query file changes | query-check |
| `custom:book` | book/** | Documentation changes | mdbook |
| `custom:xtask` | xtask/** | Build tool changes | Full rebuild |

## Local Development

```bash
# See what changed and why
cargo rail plan --merge-base --explain

# Get plan as JSON
PLAN=$(cargo rail plan --merge-base -f json)

# Extract affected crates
echo "$PLAN" | jq '.impact'

# Check which surfaces are enabled
echo "$PLAN" | jq '.surfaces | to_entries | map(select(.value.enabled)) | from_entries'

# Run xtask commands based on plan output
if echo "$PLAN" | jq -e '.surfaces["custom:queries"].enabled' > /dev/null; then
  cargo xtask query-check
fi

# Validate configuration
cargo rail config validate --strict
```

## Commands Reference

| Command | What It Does |
|---------|--------------|
| `cargo rail plan --merge-base --explain` | Show what changed and why |
| `cargo rail plan --merge-base -f json` | Get plan as JSON for scripting |
| `cargo rail plan --merge-base -f github` | Get plan as GitHub Actions outputs |
| `cargo rail config validate --strict` | Validate rail.toml configuration |
| `cargo xtask query-check` | Validate queries (run when custom:queries enabled) |
| `cargo xtask theme-check` | Validate themes (run when custom:themes enabled) |

## Expected Impact

| Scenario | Current CI | With cargo-rail |
|----------|------------|-----------------|
| Theme PR | 4-OS test matrix | **theme-check only** |
| Query PR | 4-OS test matrix | **query-check only** |
| Docs PR | 4-OS test matrix | **docgen only** |
| Code PR | All 14 crates | 2-4 affected crates |

**Estimated reduction**: 40-60% of CI runs could skip or reduce test matrix.

## Troubleshooting

### "Why did this run/skip?"

```bash
cargo rail plan --merge-base --explain
```

### "Grammar cache wasn't invalidated"

Check that `languages.toml` is in infrastructure:
```bash
cargo rail plan --merge-base -f json | jq '.surfaces.infra'
```

### "Force a full run"

```bash
cargo rail run --all --surface test
```

## Next Steps

1. **Try locally**: Install cargo-rail, run `cargo rail plan --merge-base --explain` on recent commits
2. **Evaluate**: Do the surface decisions match expected behavior?
3. **Adopt incrementally**: Start with one job, expand as confidence grows

Questions? Open an issue at https://github.com/loadingalias/cargo-rail
