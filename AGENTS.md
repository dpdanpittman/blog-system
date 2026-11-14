# AGENTS.md

Instructions for AI coding agents working in this repository.

## What this is

Exported n8n workflows for an automated blog pipeline. Five agents run in a chain: topic discovery (Hacker News and other feeds), research (SerpApi), writing and editing (a local Ollama model), and publishing to a blog API. A sixth workflow is a Telegram bot that controls the pipeline. There is no application code, build or test suite. The workflows are made by hand in the n8n editor and exported here as JSON. The last commit is from February 2026. The repo does not say whether these workflows are still the ones running.

## Layout

- `agent-1-topic-discovery.json`: scheduled topic discovery with duplicate detection.
- `agent-2-research.json`: research via SerpApi.
- `agent-3-writing.json`: writing, reads n8n Data Tables to learn from past failures.
- `agent-4-editing.json`: editing for style and length.
- `agent-5-publishing.json`: validation, SEO metadata, Data Tables logging, publish to the blog API.
- `blog-telegram-bot.json`: Telegram bot that triggers runs and reads recent posts.
- `README.md`: setup steps (Data Tables, imports, credentials, webhooks) and tuning notes.

## Commands

There is nothing to install or build. The only check is that each file is valid JSON:

```sh
for f in *.json; do python3 -m json.tool "$f" >/dev/null && echo "ok $f"; done
```

The logic lives in n8n Code nodes as JavaScript inside JSON strings. There is no linter or test for it here.

## Deployment

Workflows are imported into an n8n instance by hand, per the README. An agent does not import, activate or run workflows, and does not call the webhooks. A change to a running system is something the owner does after review.

## Rules for agents

- Ask before large changes. Explain the problem and the proposed fix first. Keep diffs small.
- Edit the JSON carefully. Keep node ids, names and `connections` consistent: connections refer to nodes by name, so renaming a node breaks its links.
- Code node source is an escaped JSON string. After editing, run the JSON check above.
- Credentials are referenced by n8n credential id and name. Never paste a key, token or password into a workflow. Secrets live in n8n credentials or environment variables (for example `$env.BLOG_API_KEY`) and a password manager.
- Never commit secrets (keys, tokens, passwords, `.env` files).
- Never delete this repository or rewrite published history.
- Do not change deployment or infrastructure details unless that is the task.
- Say plainly what you did not check. A JSON check does not prove a workflow runs.
- Commit messages: short, imperative, say what changed.

## Do not commit or publish

This repository is public.

- No secrets of any kind, including n8n credential exports.
- No new personal details about the owner, internal IP addresses, internal hostnames or chat ids. Use placeholders such as `http://OLLAMA_HOST:11434` in new or edited content.
- When exporting a workflow from n8n, check it for pinned data, execution data and hard-coded ids before committing.
