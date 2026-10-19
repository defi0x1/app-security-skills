# Multiple MRs, MRs by IID, multiple repos

GitLab commands below need `GITLAB_HOST` (e.g. `gitlab.com` or a self-managed host). GitHub: `gh pr view <n> --json baseRefName,headRefOid` for base and head SHA, `git fetch origin pull/<n>/head` to get the head objects.

Each MR is one review unit `(repo, base, head)`. Merge results in the report, not the diffs, except in strategy A.

## Fetch an MR by IID (GitLab)

Try in order, stop at the first that works. Do not install tools.

```bash
IID=123                                   # MR number (after the !)
# (a) glab: gives target_branch + full sha
glab mr view "$IID" --output json 2>/dev/null | \
  python3 -c 'import json,sys;d=json.load(sys.stdin);print(d["target_branch"],d["sha"],d["source_branch"])'

# (b) GitLab API: needs GITLAB_TOKEN (scope read_api) and the URL-encoded project path
PROJ=$(git remote get-url origin | sed -E 's#^(https?://|ssh://)?([^@/]+@)?[^/:]+[:/]##; s#\.git$##' | sed 's#/#%2F#g')   # handles https and scp-style ssh remotes
curl -sf -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://$GITLAB_HOST/api/v4/projects/$PROJ/merge_requests/$IID" | \
  python3 -c 'import json,sys;d=json.load(sys.stdin);print(d["target_branch"],d["sha"],d["source_branch"])'

# (c) Repo read access only: GitLab exposes the MR head ref, NO API needed:
git fetch -q origin "refs/merge-requests/$IID/head" && echo "HEAD_MR=$(git rev-parse FETCH_HEAD)"   # does not create a new ref
#   -> the target branch name is still needed for the base; if (a)/(b) fail, ask the user.
```

Compute the base for one MR: `BASE=$(git merge-base origin/<target_branch> <head_sha>)` (fetch the target branch first if `origin/<target_branch>` is stale). For an **already merged** MR: `head` is the merge commit or the last commit of the source branch; the base is the `merge-base` with the first parent of the merge commit (`git rev-parse <merge>^1`).

Diff of the unit: `git diff "$BASE...$HEAD"` (three dots = what the MR added since it diverged from the target).

## Strategy A: one contiguous range (only when all 3 hold)

1. All MRs are in the same **repo** and the same **target branch**.
2. **Merged in order**, with no *other* MR (outside the set) merged in between. Check:
   ```bash
   git log --merges --first-parent --oneline "$OLDEST_BASE..$NEWEST_HEAD"
   #  -> is every merge commit in the MR set? If there is a stranger MR, use strategy B.
   ```
3. The user agrees to review the "final state" instead of each MR.

In that case review **one** unit `OLDEST_BASE...NEWEST_HEAD`. You must still list each MR in the summary table (classify each MR from its own diff: `git diff <mr_base>...<mr_head> --name-only`).

## Strategy B: each MR separately (default)

For each unit `(repo, base, head)`:

Read from git objects only (no checkout or worktree, per SKILL.md principle 8):

```bash
git diff "$BASE...$HEAD"                 # diff of the MR
git show "$HEAD:$FILE"                   # file content at the MR head (replaces Read)
git grep -n "<pattern>" "$HEAD" -- .     # find callers/callees at the MR head (replaces Grep)
```

Run the review workflow (`../workflows/review.md`, MR mode, including mobile) with explicit `BASE`/`HEAD`. Multiple repos: run separately in each repo's directory, do not mix. Prefix finding IDs with the IID: `!123/SEC-001`.

After all units are done, go to `../workflows/review.md` section Multiple MRs (deduplication, combined verdict, per-MR table).

## Deduplication across MRs

Two findings are **one** when: same repo + same file + CWE of the same family (for example 862/863/285/284/639/306 are all access control; 79/80 XSS; 89/943 injection) + overlapping line ranges within ±20. Keep the version with fuller evidence or exploit path; the "MR" column lists all related IIDs.

If a later MR **fixes** a bug an earlier MR introduced (same location, a `-` hunk removes the buggy pattern), do not report it, but add one line to "Context": *"!123 introduced X, !125 fixed it."*

## Pre-existing bugs vs bugs from the MR set

Block **only** when the bug is on a line that at least one MR in the set added or modified. Verify with:

```bash
git show "$BASE:$FILE" | sed -n "${LINE_START},${LINE_END}p"   # was the pre-MR code already buggy?
git blame -L "$LINE_START,$LINE_END" "$HEAD" -- "$FILE"          # which commit introduced this line?
```

A serious pre-existing bug (HIGH+) right next to the change goes in the "Context (non-blocking)" section, with a suggestion to open a separate issue.

## Multiple repos in one MR set

The summary table gets an extra **Repo** column. Do not deduplicate across repos (different codebases). Record coverage tier per repo if they differ (for example repo A has a `go.sum` and can run trivy, repo B does not).
