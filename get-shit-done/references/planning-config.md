<planning_config>

Configuration options for `.planning/` directory behavior.

<config_schema>
```json
"planning": {
  "commit_docs": true,
  "search_gitignored": false
},
"jj": {
  "branching_strategy": "none",
  "phase_branch_template": "gsd/phase-{phase}-{slug}",
  "milestone_branch_template": "gsd/{milestone}-{slug}"
}
```

| Option | Default | Description |
|--------|---------|-------------|
| `commit_docs` | `true` | Whether to commit planning artifacts |
| `search_gitignored` | `false` | Add `--no-ignore` to broad rg searches |
| `jj.branching_strategy` | `"none"` | Bookmark strategy: `"none"`, `"phase"`, or `"milestone"` |
| `jj.phase_branch_template` | `"gsd/phase-{phase}-{slug}"` | Bookmark template for phase strategy |
| `jj.milestone_branch_template` | `"gsd/{milestone}-{slug}"` | Bookmark template for milestone strategy |
</config_schema>

<commit_docs_behavior>

**When `commit_docs: true` (default):**
- Planning files committed normally
- SUMMARY.md, STATE.md, ROADMAP.md tracked
- Full history of planning decisions preserved

**When `commit_docs: false`:**
- Skip all commits for `.planning/` files
- User must add `.planning/` to `.gitignore`
- Useful for: OSS contributions, client projects, keeping planning private

**Using gsd-tools.cjs (preferred):**

```bash
# Commit with automatic commit_docs + gitignore checks:
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" commit "docs: update state" --files .planning/STATE.md

# Load config via state load (returns JSON):
INIT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state load)
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
# commit_docs is available in the JSON output

# Or use init commands which include commit_docs:
INIT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" init execute-phase "1")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
# commit_docs is included in all init command outputs
```

**Auto-detection:** If `.planning/` is in .gitignore, `commit_docs` is automatically `false` regardless of config.json. This prevents errors when users have `.planning/` in `.gitignore`.

**Commit via CLI (handles checks automatically):**

```bash
node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" commit "docs: update state" --files .planning/STATE.md
```

The CLI checks `commit_docs` config and gitignore status internally — no manual conditionals needed.

</commit_docs_behavior>

<search_behavior>

**When `search_gitignored: false` (default):**
- Standard rg behavior (respects .gitignore)
- Direct path searches work: `rg "pattern" .planning/` finds files
- Broad searches skip gitignored: `rg "pattern"` skips `.planning/`

**When `search_gitignored: true`:**
- Add `--no-ignore` to broad rg searches that should include `.planning/`
- Only needed when searching entire repo and expecting `.planning/` matches

**Note:** Most GSD operations use direct file reads or explicit paths, which work regardless of gitignore status.

</search_behavior>

<setup_uncommitted_mode>

To use uncommitted mode:

1. **Set config:**
   ```json
   "planning": {
     "commit_docs": false,
     "search_gitignored": true
   }
   ```

2. **Add to .gitignore:**
   ```
   .planning/
   ```

3. **Existing tracked files:** If `.planning/` was previously tracked, just delete the files and commit the .gitignore change.

4. **Bookmark merges:** When using `branching_strategy: phase` or `milestone`, the `complete-milestone` workflow automatically strips `.planning/` files from staging before merge commits when `commit_docs: false`.

</setup_uncommitted_mode>

<branching_strategy_behavior>

**Bookmark Strategies:**

| Strategy | When bookmark created | Bookmark scope | Merge point |
|----------|---------------------|--------------|-------------|
| `none` | Never | N/A | N/A |
| `phase` | At `execute-phase` start | Single phase | User merges after phase |
| `milestone` | At first `execute-phase` of milestone | Entire milestone | At `complete-milestone` |

**When `jj.branching_strategy: "none"` (default):**
- All work commits to current working copy
- Standard GSD behavior

**When `jj.branching_strategy: "phase"`:**
- `execute-phase` creates a new revision and bookmark before execution
- Bookmark name from `phase_branch_template` (e.g., `gsd/phase-03-authentication`)
- All plan commits go to that bookmark
- User merges bookmarks manually after phase completion
- `complete-milestone` offers to merge all phase bookmarks

**When `jj.branching_strategy: "milestone"`:**
- First `execute-phase` of milestone creates the milestone bookmark
- Bookmark name from `milestone_branch_template` (e.g., `gsd/v1.0-mvp`)
- All phases in milestone commit to same bookmark
- `complete-milestone` offers to merge milestone bookmark to main

**Template variables:**

| Variable | Available in | Description |
|----------|--------------|-------------|
| `{phase}` | phase_branch_template | Zero-padded phase number (e.g., "03") |
| `{slug}` | Both | Lowercase, hyphenated name |
| `{milestone}` | milestone_branch_template | Milestone version (e.g., "v1.0") |

**Checking the config:**

Use `init execute-phase` which returns all config as JSON:
```bash
INIT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" init execute-phase "1")
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
# JSON output includes: branching_strategy, phase_branch_template, milestone_branch_template
```

Or use `state load` for the config values:
```bash
INIT=$(node "$HOME/.claude/get-shit-done/bin/gsd-tools.cjs" state load)
if [[ "$INIT" == @file:* ]]; then INIT=$(cat "${INIT#@file:}"); fi
# Parse branching_strategy, phase_branch_template, milestone_branch_template from JSON
```

**Bookmark creation:**

```bash
# For phase strategy
if [ "$BRANCHING_STRATEGY" = "phase" ]; then
  PHASE_SLUG=$(echo "$PHASE_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
  BOOKMARK_NAME=$(echo "$PHASE_BRANCH_TEMPLATE" | sed "s/{phase}/$PADDED_PHASE/g" | sed "s/{slug}/$PHASE_SLUG/g")
  jj new
  jj bookmark create "$BOOKMARK_NAME"
fi

# For milestone strategy
if [ "$BRANCHING_STRATEGY" = "milestone" ]; then
  MILESTONE_SLUG=$(echo "$MILESTONE_NAME" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
  BOOKMARK_NAME=$(echo "$MILESTONE_BRANCH_TEMPLATE" | sed "s/{milestone}/$MILESTONE_VERSION/g" | sed "s/{slug}/$MILESTONE_SLUG/g")
  jj new
  jj bookmark create "$BOOKMARK_NAME"
fi
```

**Merge options at complete-milestone:**

| Option | Command | Result |
|--------|---------|--------|
| Squash merge (recommended) | `jj squash --from BOOKMARK` | Single clean commit per bookmark |
| Merge with history | `jj new main BOOKMARK` | Preserves all individual commits |
| Delete without merging | `jj bookmark delete BOOKMARK` | Discard bookmark work |
| Keep bookmarks | (none) | Manual handling later |

Squash merge is recommended — keeps main history clean while preserving the full development history.

**Use cases:**

| Strategy | Best for |
|----------|----------|
| `none` | Solo development, simple projects |
| `phase` | Code review per phase, granular rollback, team collaboration |
| `milestone` | Release branches, staging environments, PR per version |

</branching_strategy_behavior>

</planning_config>
</output>
