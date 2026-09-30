# Tour as a PR/MR comment

The comment form of the tour. It keeps what a comment can carry: the reading
order, a checkbox per file and per route, the file links into the diff, the
risk badges as text, and the dev-server instructions. It drops what a comment
cannot: the scatter, the landscape, per-reviewer tick sync (the host stores
the task-list state itself, shared by everyone who can tick), and the base
URL input.

Render it from the same tour JSON as the page. GitHub and GitLab both turn
`- [ ]` into a live checkbox in comments. GitHub caps a comment at 65,536
characters; above ~40 files, wrap each chapter's list in `<details>` (the
first chapter open) and drop `look_for` lines that carry no smell or hotspot.

File links use the anchors from `hosts.md`, computed in the shell:

```bash
anchor() { # github
  printf '%s/files#diff-%s' "$PR_URL" "$(printf %s "$1" | shasum -a 256 | cut -c1-64)"
}
anchor() { # gitlab
  printf '%s/diffs#%s' "$PR_URL" "$(printf %s "$1" | shasum -a 1 | cut -c1-40)"
}
```

Append `R<line>` (GitHub) or `_<old>_<new>` (GitLab) when the file carries
`line`.

## Template

Start with the marker so a re-run can find and edit the comment instead of
posting a second one.

```markdown
<!-- jr-rails-pr-tour -->
## Reading order

<summary, verbatim from `summary`>

Risk badges: <risk_source>. Tick a file when you have read it. <If an
artifact was also published: "The interactive tour is here: <link>.">

### 01 The ImportRun record

<lede>

- [ ] [db/migrate/20260901120000_create_import_runs.rb](<anchor>) `migration` `added` +18 −0
  Creates the import_runs table with a status column and counters. All files that follow use these columns.
  *Look for:* The migration has no down method. Make sure that it is reversible.
- [ ] [app/models/import_run.rb](<anchor>R58) `model` `added` +96 −0 **reworked** 71 · 7/9 commits · +1 smell
  Defines ImportRun and its status changes. The job, the controller, and the progress view use it.
  *Look for:* The method advance! has a flog score of 41 (line 58). Adds FeatureEnvy in ImportRun#advance! (line 58).

### 02 The background job

…

### Safe to skim

<lede with the counts>

<details>
<summary>24 files</summary>

- [ ] [app/views/products/_row.html.erb](<anchor>) rename only
- [ ] …

</details>

### Left out of the tour

<details>
<summary>3 files</summary>

- `db/schema.rb` generated from migrations
- `Gemfile.lock` lockfile

</details>

## Try it on the dev server

<qa.start, verbatim if it is a sentence, else "Start it with `bin/dev` on the PR branch.">
<If migrations: "The PR has N migrations; the server runs them on your development database.">
Base URL: `http://localhost:3000`

- [ ] `GET /import_runs/new` (ImportRuns#new, ch. 03) The upload form. Choose a CSV file and submit it.
- [ ] `POST /import_runs` via `/import_runs/new` (ImportRuns#create, ch. 03) Submit the form. The response redirects to the progress page.
- [ ] `GET /import_runs/1` (`/import_runs/:id`, ImportRuns#show, ch. 03) The counters increase while the job runs.
```

Rules that differ from the page:

- Badges are text: `**label**` then the numbers; the label set is the same.
- Routes are code, not links: a comment on github.com linking to
  `http://localhost:3000` is a link into the reader's own machine, which
  looks like a mistake to anyone reading on a phone. The concrete path goes
  first, the pattern in parentheses after it.
- Chapter numbers stay (the sequence is the point). No emoji as markers.
- Same prose, same Simplified Technical English; nothing is rewritten for
  the comment.

## Posting and editing

```bash
gh pr comment $PR --body-file tour.md
glab mr note $PR --message "$(cat tour.md)"
```

Re-run: find the comment by the marker and edit it in place, so ticked
boxes people set on the host are kept where the file list did not change.

```bash
ID=$(gh api "repos/{owner}/{repo}/issues/$PR/comments" --paginate --jq '.[] | select(.body | startswith("<!-- jr-rails-pr-tour -->")) | .id' | head -1)
[ -n "$ID" ] && gh api "repos/{owner}/{repo}/issues/comments/$ID" -X PATCH -F body=@tour.md || gh pr comment $PR --body-file tour.md

ID=$(glab api "projects/:id/merge_requests/$PR/notes" --paginate | jq -r '.[] | select(.body | startswith("<!-- jr-rails-pr-tour -->")) | .id' | head -1)
[ -n "$ID" ] && glab api "projects/:id/merge_requests/$PR/notes/$ID" -X PUT -f body=@tour.md || glab mr note $PR --message "$(cat tour.md)"
```

Editing replaces the body, so a re-run resets ticks for files whose line
changed; say so in the summary line when it happens ("re-run after new
commits; ticks reset for 3 files").
