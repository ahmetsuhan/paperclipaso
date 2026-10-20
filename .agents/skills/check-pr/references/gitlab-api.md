# GitLab API Reference

Useful GitLab REST API calls for working with merge request discussions, using `glab api`.

`glab api` automatically resolves `:fullpath` to the URL-encoded project path from the local git remote.

## Fetch MR details

```bash
glab mr view <MR_IID> --output json
```

Key fields (compared to GitHub equivalents):
- `iid` - internal MR number (use this, not `id`)
- `source_branch` - equivalent to GitHub's `headRefName`
- `sha` - HEAD commit SHA, equivalent to GitHub's `headRefOid`
- `description` - equivalent to GitHub's `body`

## Fetch all discussions (inline + general comments)

```bash
glab api "projects/:fullpath/merge_requests/<MR_IID>/discussions?per_page=100"
```

Paginate with `&page=2`, `&page=3`, etc. until the response array length is less than `per_page`.

Each discussion object:
- `id` - discussion ID (used for resolution)
- `resolved` - `true` or `false`
- `notes` - array of note objects

Each note object:
- `type` - `"DiffNote"` for inline diff comments, `null` for general comments
- `author.username` - author's username
- `body` - comment text
- `position.new_path` - file path (for `DiffNote` type)

## Filter for unresolved inline diff comments

```bash
glab api "projects/:fullpath/merge_requests/<MR_IID>/discussions?per_page=100" | \
  jq '[.[] | select(.resolved == false and (.notes[0].type == "DiffNote"))]'
```

## Resolve a single discussion

```bash
glab api --method PUT \
  "projects/:fullpath/merge_requests/<MR_IID>/discussions/<DISCUSSION_ID>" \
  --field resolved=true
```

There is no batch resolution in GitLab - issue one PUT per discussion.

## Fetch pipeline status for an MR

```bash
glab api "projects/:fullpath/merge_requests/<MR_IID>/pipelines"
```

Pipeline statuses: `running`, `pending`, `success`, `failed`, `canceled`, `skipped`.

## Fetch jobs for a specific pipeline

```bash
glab api "projects/:fullpath/pipelines/<PIPELINE_ID>/jobs"
```

Each job has `name`, `status`, `stage`, and `web_url`.

## Fetch MR notes (general comments and bot reviews)

```bash
glab api "projects/:fullpath/merge_requests/<MR_IID>/notes?per_page=100"
```

Filter by `author.username` to find Greptile bot comments. The exact bot username depends on the Greptile installation - check the first Greptile comment to identify it.

## Post a comment on an MR

```bash
glab mr note <MR_IID> --message "your message here"
```

Or via API:

```bash
glab api --method POST \
  "projects/:fullpath/merge_requests/<MR_IID>/notes" \
  --field body="your message here"
```

## Greptile freshness gate for MRs

For GitLab installations with Greptile integration, apply the same freshness rule against the MR's current head SHA:

```bash
# 1. Get the MR's current head SHA
MR_SHA=$(glab mr view <MR_IID> --output json | jq -r '.sha // .diff_refs.head_sha')

# 2. Find the latest pipeline for that EXACT sha, then the Greptile job within it.
LATEST_PIPELINE_ID=$(glab api "projects/:fullpath/merge_requests/<MR_IID>/pipelines" \
  | jq -r --arg sha "$MR_SHA" '[.[] | select(.sha == $sha)] | sort_by(.id) | last | .id // empty')

if [ -n "$LATEST_PIPELINE_ID" ]; then
  GREPTILE_JOBS=$(glab api "projects/:fullpath/pipelines/$LATEST_PIPELINE_ID/jobs" \
    | jq --arg sha "$MR_SHA" '[.[] | select(.name | test("greptile"; "i"))
        | {name, status, pipeline_sha: $sha}]')
else
  GREPTILE_JOBS='[]'
fi

GREPTILE_JOB_SUCCESS=$(echo "$GREPTILE_JOBS" \
  | jq '[.[] | select(.status == "success")] | length')
GREPTILE_JOB_BLOCKING=$(echo "$GREPTILE_JOBS" \
  | jq '[.[] | select(.status != "success")] | length')

# 3. If Greptile integrates via MR notes instead of a CI job, require the newest
#    Greptile note to reference the current head sha and report a clean review.
GREPTILE_NOTES=$(glab api "projects/:fullpath/merge_requests/<MR_IID>/discussions?per_page=100" \
  | jq --arg sha "$MR_SHA" '[.[].notes[]
      | select(.author.username | test("greptile"; "i"))]
      | sort_by(.updated_at // .created_at)')
GREPTILE_NOTE_CLEAN=$(echo "$GREPTILE_NOTES" \
  | jq --arg sha "$MR_SHA" 'if length == 0 then 0
      elif ((last.body // "") | contains($sha) and test("Confidence Score:[[:space:]]*5/5|Confidence:[[:space:]]*5/5|\\b5/5\\b"; "i") and (test("Prompt To Fix|blocking issue|failed|action required"; "i") | not)) then 1
      else 0 end')

if [ "$GREPTILE_JOB_SUCCESS" = "0" ] && [ "$GREPTILE_NOTE_CLEAN" = "0" ]; then
  echo "Blocked: no successful Greptile job or completed-clean current-head Greptile note is tied to MR head $MR_SHA."
  exit 1
fi

if [ "$GREPTILE_JOB_BLOCKING" != "0" ]; then
  echo "Blocked: at least one Greptile job for MR head $MR_SHA did not succeed."
  exit 1
fi
```

Block completion if, for `MR_SHA`, there is (a) no Greptile job or note at all (missing), (b) the newest Greptile job/note is tied to a different sha (stale), or (c) the Greptile job status is not `success`. A completed Greptile result for a different SHA is stale and must block completion.
