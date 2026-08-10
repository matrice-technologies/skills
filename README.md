# Fixit skills

Agent skills for [Fixit](https://fixit.matricetechnologies.com) — product
feedback captured from the people actually using your app, with the console
output, failed requests, a screenshot and a DOM replay attached.

## Install

```bash
npx skills add matrice-technologies/skills
```

Works with Claude Code, Cursor, Codex, OpenCode and the rest of the agents the
[`skills`](https://github.com/vercel-labs/skills) installer supports.

## What you can then do

| Ask | What happens |
| --- | --- |
| `/fixit setup` | Installs the CLI, walks you through login in your browser, asks which project this repository is, and puts the widget where your stack loads third-party scripts |
| `/fixit the last 3 reports from truva` | Answers from the inbox — no code touched |
| `/fixit truva/40` | Reads the report, claims it, fixes the cause, verifies, releases it |

## The CLI on its own

The skill drives [`@matrice-technologies/fixit-cli`](https://www.npmjs.com/package/@matrice-technologies/fixit-cli),
which is useful by hand too:

```bash
npm install -g @matrice-technologies/fixit-cli
fixit login
fixit list
```

## Contributing

`skills/fixit/SKILL.md` here is published from the Fixit repository; edit it
there.
