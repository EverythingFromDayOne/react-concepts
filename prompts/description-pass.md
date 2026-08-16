# Cross-repo prompt — add `description` frontmatter

**This prompt runs in a CORPUS repo, not in `corpus-web`.** Run it once per repo, on a
branch, one PR each. Do not batch repos.

Applies to: `nextjs-concepts`, `react-concepts`, `angular-concepts`, `nestjs-concepts`.

`dsa-concepts` needs this too but has no GitHub remote yet — run it locally once published,
before its first tag. Does **not** apply to `demo-auth-concepts`, `demo-authz-concepts`, or
`demo-attacked-web`: those are runnable demo apps with no articles.

---

## Why

`corpus-web` renders a one-line dek under every H1 and reuses it as the page's meta
description. No corpus carries that field. Deriving it from the first paragraph was
considered and rejected — roughly a third of articles open with a callout, and the derived
text reads badly as a search result.

The field is **required**. An article without one fails the site build. **196 articles
across the four corpora are missing it**, and four verify gates in `corpus-web` are red
for exactly this reason (tracked as issue #3).

## What the frontmatter actually looks like — read this before editing

Established by the session 2 audit, against real files. Do not assume otherwise.

- **There is no `title` key.** Every article's title comes from the body's first H1.
  So: read the H1 to know what the article is called.
- **`status` is sometimes an object, not a string** — confirmed in `react-concepts`,
  `nestjs-concepts`, and some `angular-concepts` recipes. Do not touch, flatten, or
  reformat it. A careless edit to surrounding indentation corrupts it silently.
- **`react-concepts` and `nestjs-concepts` have no `docs/` wrapper.** Articles sit at a
  different depth than in the other two. Walk the tree; do not assume a path shape.
- **14 `react-concepts` articles have no title at all** — no frontmatter H1, no body H1.
  These are tracked as Debt D11 in `corpus-web`. **Skip them.** Naming an untitled article
  is authoring, not a mechanical pass.

## Scope — read this twice

**Add exactly one frontmatter key per article. Change nothing else.**

- Insert `description` **immediately after the `article_id` / `recipe_id` line.** If an
  article has neither key, skip it and add it to the "needs author" list.
- Do NOT reword, restructure, retitle, or reformat any prose.
- Do NOT touch code blocks, `related`, `status`, or any other frontmatter key.
- Do NOT reorder frontmatter keys.
- Do NOT create a frontmatter block where none exists. Report it instead.
- Do NOT add the field to `README.md`, `roadmap.md`, `progress.md`, or any non-article file.

A diff on this branch should show, per file, exactly one added line.

## Writing the dek

One sentence. 90–160 characters. Sentence case. No trailing period unless it is genuinely
two clauses.

It must say **what the article establishes**, not what topic it covers. The test: could
this sentence sit under a search result and tell a reader whether to click?

Draw it from the article's own opening — the H1 plus the lead callout or first paragraph
usually states the thesis already. **Do not invent a claim the article does not make.** If
you cannot write one without asserting something unsupported, leave the field out, add the
path to the "needs author" list, and move on. A wrong dek is worse than a missing one: the
build failure is loud, a false claim is silent.

Good:

```yaml
article_id: how-react-renders
description: Render and commit are two phases, and most performance advice conflates them
```

```yaml
article_id: cache-components
description: Next 16 inverted the caching defaults, which makes most existing guidance wrong
```

Bad, and why:

- `Learn about how React renders` — restates the title, says nothing
- `A comprehensive deep dive into the React rendering pipeline` — marketing
- `Everything you need to know about caching` — a claim about the article, not the subject
- `React renders in a single pass for performance` — invented, and false

## Procedure

1. Branch `chore/add-description-frontmatter`.
2. Walk every article file in the repo. For each, read the frontmatter, the H1, and the
   first ~15 lines of body.
3. Write the dek. Insert it after the id key.
4. Emit a table in your session output: path, H1 title, dek, character count.
5. Emit a separate **"needs author"** list for every article you skipped, with the reason
   — untitled, no frontmatter, no id key, or thesis unclear.
6. Run the repo's existing gates. They must all still pass. Nothing here should affect code
   blocks or links; if a gate breaks, the change went beyond scope.
7. Follow that repo's own session-close protocol. Open a PR. **Do not merge.**

## Verification before opening the PR

```bash
git diff --stat                    # every file: 1 insertion, 0 deletions
git diff | grep '^-' | grep -v '^---'   # must return nothing
```

Any deletion means the pass went out of scope. Reset that file and redo it.

Then confirm no YAML was corrupted — this is the failure mode that matters most, because
`status` objects break silently:

```bash
node -e "const fs=require('fs'),m=require('gray-matter');let bad=0;
for(const f of process.argv.slice(1)){try{m(fs.readFileSync(f,'utf8'))}catch(e){bad++;console.log('BROKEN',f,e.message)}}
console.log(bad?'FAIL':'all frontmatter parses')" $(git diff --name-only)
```

## Afterwards

Tag the corpus repo. Then, in `corpus-web`, run `/promote-content` to bump that submodule.
Separate PRs in separate repos, and they stay that way.
