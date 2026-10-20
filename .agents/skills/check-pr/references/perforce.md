# Perforce Reference

Perforce-specific commands for each `check-pr` step. Use these in place of the
GitHub/GitLab commands when step 0 detects `VCS="perforce"` (or `--vcs perforce`).

## 1. Identify the CL

```bash
# List pending changelists for the current user/client
p4 changes -s pending -u $P4USER -c $P4CLIENT
```

- Perforce: changelist number (CL), `shelved` files for in-review CLs

## 2. Fetch CL details

```bash
# Get changelist description, files, and status
p4 describe -s <CL_NUMBER>

# Get shelved files (for in-review CLs)
p4 describe -S <CL_NUMBER>

# Get the diff of the shelved changelist
p4 diff2 //...@=<CL_NUMBER> //...@=<CL_NUMBER>

# List review comments (if using p4 review workflow)
p4 review -c <CL_NUMBER>
```

Key Perforce CL fields:
- `Change`: changelist number
- `Status`: `pending`, `submitted`, `shelved`
- `Description`: the CL description / commit message
- `Files`: list of files in the CL

## 3. Wait for pending checks

**Perforce:** Perforce doesn't have built-in CI checks natively. If the team uses a review tool (Swarm, etc.) or an external CI triggered by shelve events, check the relevant system. Otherwise, proceed to analysis immediately.

## 4. Greptile freshness

For Perforce installations with Greptile integration, apply the same rule using the CL's current shelved-revision identity and the Greptile webhook/review artifact tied to it; a Greptile result for an earlier shelf is stale and must block completion.

## 5. Analyze the CL

- **Perforce:** review comments from `p4 review` or external review tools
- **Perforce:** CL description should include a clear summary, affected files rationale, and testing notes

## 8. Fix issues

Ensure files are open in the correct CL, then open files for edit, make changes, and re-shelve:

```bash
p4 edit <file>
# make changes
p4 shelve -f -c <CL_NUMBER>
```

## 9. Resolve review threads

**Perforce** - Perforce does not have a native "resolve thread" concept. Instead, mark comments as addressed by updating the CL description or by responding in the review tool being used (Swarm, etc.). If using `p4 review`:

```bash
# Mark files as reviewed after addressing feedback
p4 review -c <CL_NUMBER>
```

## 10. Multiple CLs

To check multiple changelists at once:
```bash
p4 changes -s pending -u $P4USER -c $P4CLIENT -l
```

Status checks summary in the output is N/A for Perforce.
