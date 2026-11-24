# AGENTS.md

Instructions for AI coding agents working in this repository.

## What this is

Scrubbed exports of the n8n workflows behind an automated blog (v2, November 2026). Five agent workflows run in a chain:
topic discovery, research (SerpApi), writing (local Ollama model), editing (an external editor service, with the local
model as fallback) and publishing to a blog API. Around them: a weekly digest, an error alert workflow with retries, an
X poster and a daily ham radio band report built from NOAA data with fixed rules. There is no application code, build or
test suite. The README explains each workflow, the Data Tables and how to import them.

The workflows are built and run in a private setup. This repo is a public copy. Every instance id, credential, chat id,
host and address in it is a placeholder.

## Layout

- `workflows/agent-1-topic-discovery.json`: daily topic pick from a queue or public feeds.
- `workflows/agent-2-research.json`: source page, SerpApi search, fetched results, research summary.
- `workflows/agent-3-writer.json`: writes the draft, reads the learning Data Tables first.
- `workflows/agent-4-editor.json`: hands the draft to an external editor, falls back to the local model.
- `workflows/agent-5-publisher.json`: quality checks, Publish Gate, publish, Telegram note, learning tables, X hand-off.
- `workflows/blog-weekly-digest.json`: Sunday digest post.
- `workflows/blog-pipeline-error-alerts.json`: error workflow for all of the above.
- `workflows/blog-post-to-x.json`: posts one line and the link on X.
- `workflows/ham-radio-band-conditions-daily.json`: daily band report, no model.
- `README.md`: what each workflow does, Data Tables, credentials, placeholders, import steps.
- v1 (the first hand-made version and the Telegram command bot) is only in git history, commit `14fbe8f` and earlier.

## Commands

There is nothing to install or build. The only check is that each file is valid JSON:

```sh
for f in workflows/*.json; do python3 -m json.tool "$f" >/dev/null && echo "ok $f"; done
```

The logic lives in n8n Code nodes as JavaScript inside JSON strings. There is no linter or test for it here.

## Deployment

Workflows are imported into an n8n instance by hand, per the README. An agent does not import, activate or run
workflows, and does not call the webhooks. A change to a running system is something the owner does after review.

## Rules for agents

- Ask before large changes. Explain the problem and the proposed fix first. Keep diffs small.
- Edit the JSON carefully. Connections refer to nodes by name, so renaming a node breaks its links.
- Code node source is an escaped JSON string. After editing, run the JSON check above.
- Never paste a key, token or password into a workflow. Secrets live in n8n credentials or environment variables (for
  example `$env.BLOG_API_KEY`).
- Never commit secrets (keys, tokens, passwords, `.env` files).
- Never delete this repository or rewrite published history.
- Say plainly what you did not check. A JSON check does not prove a workflow runs.
- Commit messages: short, imperative, say what changed.

## Do not commit or publish

This repository is public.

- No secrets of any kind, including n8n credential exports.
- No personal details about the owner, home location, internal IP addresses, internal hostnames, tailnet names, n8n
  instance URLs, chat or topic ids, email addresses, or credential, webhook, Data Table or workflow ids from a real
  instance. Use the placeholders listed in the README (`http://OLLAMA_HOST:11434`, `https://your-n8n.example`,
  `https://your-blog.example`, `{{ DATA_TABLE: name }}`, `{{ CREDENTIAL: ... }}`, `YOUR_TELEGRAM_CHAT_ID`).
- When refreshing an export from n8n, scrub it the same way and check it for pinned data, execution data and
  hard-coded ids before committing. Then grep the whole tree for addresses, hostnames and ids before you push.
