---
name: fixit
description: Set up Fixit in a repository and work the feedback it collects. Installs the widget into this codebase, answers questions about incoming reports, and takes a report from received to released. Use when the user runs /fixit setup, names a report like truva/40, asks what users have reported, or asks to install the Fixit feedback widget.
---

# Fixit

Fixit collects feedback from the people actually using an app. The widget sits in
the corner of the page; a user points at what is wrong, draws on it, records a
few seconds of the session, and sends. What arrives is a report carrying the
console output, the failed network requests, a screenshot, a DOM replay, and the
exact element they pointed at.

This skill does three things: sets a repository up, answers questions about the
reports, and fixes them.

## The `fixit` CLI

Everything here goes through the CLI. Check it is present before anything else:

```bash
fixit --version
```

If that fails, install it:

```bash
npm install -g @matrice-technologies/fixit-cli
```

Then check the machine is connected:

```bash
fixit whoami
```

If that prints a host, it is connected and you can go straight to the work. If
it says `Not connected`, log in — see below. Credentials live in
`~/.fixit/config.json`, outside the repository, so nothing needs gitignoring.

### Logging in

Login happens in the user's browser, so it takes two commands. **Never run bare
`fixit login`** — it blocks polling for an approval the user has not been shown
yet, and the session will hang.

```bash
fixit login --begin
```

This prints a short code and a URL, then exits. Show the user the URL and ask
them to open it and approve. On that screen they choose which workspace — and
optionally which project — this machine may touch.

When they say they have approved it:

```bash
fixit login --finish
```

This blocks only until the approval lands, then saves the token. If it reports
the login expired, start again with `--begin`.

---

## Mode 1 — `/fixit setup`

Connect this repository to a Fixit project and install the widget in it.

### 1. Connect

Work through the section above: CLI installed, `fixit whoami` reporting a host.

### 2. Choose the project

```bash
fixit projects
```

Show the user the list and **ask which project this repository is**. Do not
guess from the directory name — one repository often serves a project named
something else entirely. If exactly one project comes back, confirm it rather
than assuming.

If the right project does not exist yet, say so: it has to be created in the
dashboard first.

### 3. Get the snippet

```bash
fixit widget --project <slug>
```

This prints the exact `<script>` tag for that project, with the right key and
the right host. Use what it prints — do not assemble a URL by hand.

**Copy `src` across verbatim. It is absolute, and it has to stay absolute** —
do not shorten it to `/widget.js` because that looks tidier. The widget is
served by the Fixit instance, not by the site it runs on, so a relative path
404s on any other domain. Worse, the widget derives the API it posts to from
its own `src`: relativise it and every report is sent to the host page's own
origin instead of Fixit, which fails silently.

Read its second line carefully. If the project has a **domain** set, feedback is
only accepted from that domain: the widget will load on `localhost` but every
submission will be refused. Tell the user this before they test locally.

### 4. Install it

Work out what this repository is by looking, not by guessing — check
`package.json`, the framework config, and the directory layout. Then put the
widget where that stack loads third-party scripts:

| Stack | Where |
| --- | --- |
| Next.js App Router | `app/layout.tsx`, inside `<body>`, via `next/script` |
| Next.js Pages Router | `pages/_document.tsx`, before `</body>` |
| Vite / CRA / any SPA | `index.html`, before `</body>` |
| Astro | the shared layout in `src/layouts/` |
| SvelteKit | `src/app.html` |
| Nuxt | `app.vue`, or `app.head.script` in `nuxt.config` |
| Plain HTML | before `</body>` on each page, or the shared include |

For Next.js App Router that means:

```tsx
import Script from "next/script"

// inside <body>
<Script
  src="https://fixit.matricetechnologies.com/widget.js"
  data-fixit-key="pk_..."
  strategy="afterInteractive"
/>
```

Everywhere else the printed tag goes in as-is.

The key is **public** — it ships in the page source of every site running the
widget, and the domain check is what protects the project. So hardcoding it is
fine. Only move it to an environment variable if that is already this
repository's convention for such things.

If the user wants the widget on staging but not in production, gate it on
whatever environment flag this repository already uses, and say which one you
used.

Before moving on, read back what you wrote and check the `src` still starts with
`https://` and matches what `fixit widget` printed.

### 5. Pin the project

So later commands do not need `--project`, write `.env.fixit` in the repository
root:

```
FIXIT_PROJECT=<slug>
```

Add `.env.fixit` to `.gitignore` if it is not already there.

### 6. Report back

Tell the user, briefly: which project this repository is now connected to, which
file the widget went into, and whether the domain setting will let them test
locally. Then show them what they can do next — `/fixit what came in today`, or
`/fixit <slug>/<number>` to fix one.

---

## Mode 2 — questions about feedback

When the user asks something in prose — "what came in this week", "the last 3
feedbacks from project truva", "is anyone complaining about checkout" — turn it
into a `fixit list` and answer from the result. Do not start fixing anything.

```bash
fixit list                          # everything, newest first, across all projects
fixit list --project truva -n 3     # last 3 from one project
fixit list --status received        # not yet picked up
fixit list --kind broken            # broken | suggestion | confusing | other
fixit list --search checkout        # matches title or note
```

Listing is newest first, so "the last 3" is `-n 3`. With no `--project` it spans
every project the token can reach.

Answer in prose, with the reference (`truva/40`) for anything worth acting on.
Use `fixit show <ref>` when the user asks about one report specifically, or when
the list alone cannot answer the question. Add `--json` when you need to parse
rather than read.

---

## Mode 3 — fix a report

`/fixit truva/40`, or "fix the checkout one". A report is named `project/number`.
A bare `40` works when `.env.fixit` pins a project.

If the user has not said which, list what is waiting and ask:

```bash
fixit list --status received
```

### Read it

```bash
fixit show truva/40
```

Read the whole thing before touching code. What matters most:

- **the note** — what they expected, and what happened instead
- **element selector** — the DOM node they pointed at; your best lead for finding
  the component in source
- **console errors** — often name the failing file and line outright
- **failed requests** — a 4xx or 5xx here usually means the bug is server-side,
  not in the component they were looking at
- **page URL** — maps to a route in this repository
- **session replay** — gzipped rrweb events; gunzip and read as JSON when you
  need the sequence of DOM changes that led to the failure

### Claim it

Before editing anything, so nobody duplicates the work:

```bash
fixit claim truva/40 -m "Picking this up"
```

### Fix the cause

Search using the selector, the visible text, the route from the URL, and any
file named in a stack trace. Fix the **cause**, not the symptom: if a console
error names a null dereference, work out why the value is null rather than
adding an optional chain and moving on.

Match the conventions of the code you are editing. Read `CLAUDE.md` or
`AGENTS.md` if this repository has one.

### Verify

Run whatever this repository provides — typecheck, lint, tests, build. Do not
skip it. Releasing an unverified change is worse than leaving the report open.

If you cannot reproduce it or cannot fix it, **leave it claimed** and say what
you found:

```bash
fixit comment truva/40 "Could not reproduce: ..."
```

Then tell the user. Never release something you did not fix.

### Release

```bash
fixit release truva/40 -m "Fixed in <file>: <what changed>"
```

Write that comment for a person reading the inbox later: name the file, describe
the change in one line.

> Check first if "released" means *deployed* to this user. This marks the report
> released as soon as the fix is verified locally. If their process only calls
> something released once it ships, do this step after the deploy instead.

### Report back

What was wrong, what you changed, and that the report is released. A few lines.

### Several at once

Do them one at a time, through the full cycle. Do not claim a batch up front — a
claim signals active work, and claiming five while fixing one misleads everyone
looking at the board.

---

## Command reference

```bash
fixit login --begin / --finish   # connect this machine (see above)
fixit whoami                     # what it is connected to
fixit projects                   # projects this token can reach
fixit widget [-p <slug>]         # the widget snippet for a project
fixit list [options]             # newest first; no --project spans the workspace
fixit show <ref>                 # full detail with logs, screenshot, replay
fixit claim <ref> [-m "note"]    # -> being fixed
fixit release <ref> [-m "note"]  # -> released
fixit status <ref> <status>      # received | being_fixed | released
fixit comment <ref> "text"       # comment without changing status
```

Options: `-p/--project`, `-s/--status`, `-k/--kind`, `-q/--search`, `-n/--limit`,
`-m/--message`, `--host`, `--json`.

Credentials resolve in this order: `FIXIT_TOKEN` in the environment, then
`.env.fixit` searched upwards from the working directory, then
`~/.fixit/config.json` from `fixit login`.
