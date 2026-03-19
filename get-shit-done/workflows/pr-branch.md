<purpose>
Create a clean bookmark for pull requests by filtering out .planning/ changes.
The PR bookmark contains only code changes — reviewers don't see GSD artifacts
(PLAN.md, SUMMARY.md, STATE.md, CONTEXT.md, etc.).

Uses jj restore with path filtering to rebuild a clean history.
</purpose>

<process>

<step name="detect_state">
Parse `$ARGUMENTS` for target bookmark (default: `main`).

```bash
CURRENT_BOOKMARK=$(jj log -r @ --no-graph -T 'bookmarks')
TARGET=${1:-main}
```

Check preconditions:
- Must be on a feature bookmark (not main/master)
- Must have commits ahead of target

```bash
AHEAD=$(jj log -r 'TARGET..@' --no-graph | wc -l)
if [ "$AHEAD" = "0" ]; then
  echo "No commits ahead of $TARGET — nothing to filter."
  exit 0
fi
```

Display:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► PR BRANCH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Bookmark: {CURRENT_BOOKMARK}
Target: {TARGET}
Commits: {AHEAD} ahead
```
</step>

<step name="analyze_commits">
Classify commits:

```bash
# Get all commits ahead of target
jj log -r 'TARGET..@' --no-graph
```

For each commit, check if it ONLY touches .planning/ files:

```bash
# For each commit hash
FILES=$(jj diff -r $HASH --summary)
ALL_PLANNING=$(echo "$FILES" | grep -v "^\.planning/" | wc -l)
```

Classify:
- **Code commits**: Touch at least one non-.planning/ file → INCLUDE
- **Planning-only commits**: Touch only .planning/ files → EXCLUDE
- **Mixed commits**: Touch both → INCLUDE (planning changes come along)

Display analysis:
```
Commits to include: {N} (code changes)
Commits to exclude: {N} (planning-only)
Mixed commits: {N} (code + planning — included)
```
</step>

<step name="create_pr_branch">
```bash
PR_BRANCH="${CURRENT_BOOKMARK}-pr"

# Create PR bookmark from target
jj new "$TARGET"
jj bookmark create "$PR_BRANCH" -r @
```

Restore only code commits (in order):

```bash
for HASH in $CODE_COMMITS; do
  jj new @
  jj restore --from "$HASH"
  # Remove any .planning/ files that came along in mixed commits
  rm -rf .planning/ 2>/dev/null || true
  jj describe -m "$(jj log -r $HASH --no-graph -T description)"
done
```

Return to original bookmark:
```bash
jj new "$CURRENT_BOOKMARK"
```
</step>

<step name="verify">
```bash
# Verify no .planning/ files in PR bookmark diff
PLANNING_FILES=$(jj diff --from "$TARGET" --to "$PR_BRANCH" --summary | grep "^\.planning/" | wc -l)
TOTAL_FILES=$(jj diff --from "$TARGET" --to "$PR_BRANCH" --summary | wc -l)
PR_COMMITS=$(jj log -r 'TARGET..PR_BRANCH' --no-graph | wc -l)
```

Display results:
```
PR bookmark created: {PR_BRANCH}

Original: {AHEAD} commits, {ORIGINAL_FILES} files
PR bookmark: {PR_COMMITS} commits, {TOTAL_FILES} files
Planning files: {PLANNING_FILES} (should be 0)

Next steps:
  jj git push --bookmark {PR_BRANCH}
  gh pr create --base {TARGET} --head {PR_BRANCH}

Or use /gsd:ship to create the PR automatically.
```
</step>

</process>

<success_criteria>
- [ ] PR bookmark created from target
- [ ] Planning-only commits excluded
- [ ] No .planning/ files in PR bookmark diff
- [ ] Commit messages preserved from original
- [ ] User shown next steps
</success_criteria>
</output>
