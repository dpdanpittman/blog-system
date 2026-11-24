# Automated blog (n8n), v2

Exported n8n workflows for a blog that writes itself. Five agent workflows find a topic, research it, write a draft, edit it
and publish it, once a day. Around them sit a weekly digest, an error alert workflow, an X poster and a daily ham radio band
report built from NOAA space weather data with fixed rules.

These exports match the workflows that run in production as of November 2026. They were scrubbed before publishing:
every instance id, credential, chat id, host and address is a placeholder. See [Placeholders](#placeholders).

Everything the pipeline publishes is written by AI. The site it runs for says so on every page.

## What runs, and when

Times are UTC (the workflow timezone). Change it in each workflow's settings and in the Code nodes that call
`setZone('UTC')`.

| File                                             | Workflow name in n8n                         | Trigger                              | What it does                 |
| ------------------------------------------------ | -------------------------------------------- | ------------------------------------ | ---------------------------- |
| `workflows/agent-1-topic-discovery.json`         | PHASE 5: Agent 1 (Automated Topic Discovery) | daily 13:00, or another workflow     | Picks one topic              |
| `workflows/agent-2-research.json`                | PHASE 5: Agent 2 (With Search)               | webhook from Agent 1                 | Searches and reads sources   |
| `workflows/agent-3-writer.json`                  | PHASE 5: Agent 3 (With Data Tables)          | webhook from Agent 2                 | Writes the draft             |
| `workflows/agent-4-editor.json`                  | PHASE 5: Agent 4 (With Ollama)               | webhook from Agent 3                 | Edits the draft              |
| `workflows/agent-5-publisher.json`               | PHASE 5: Agent 5 (With Publishing)           | webhook from Agent 4 or the digest   | Checks, publishes, announces |
| `workflows/blog-weekly-digest.json`              | Blog Weekly Digest                           | Sundays 10:00                        | A digest post of the week    |
| `workflows/blog-pipeline-error-alerts.json`      | Blog Pipeline Error Alerts                   | error workflow of all the others     | Alerts, marks runs, retries  |
| `workflows/blog-post-to-x.json`                  | Blog Post to X                               | webhook from Agent 5 and the ham job | One post on X per item       |
| `workflows/ham-radio-band-conditions-daily.json` | Ham Radio Band Conditions (daily)            | daily 06:00                          | Band report, no model        |

The agents hand off over webhooks. Each webhook answers at once, so one agent does not wait on the next. A `run` object
(run id, mode, day, dry run flag) travels with the payload from Agent 1 to Agent 5.

## The five agents

### Agent 1: topic discovery

- Reads `blog_system_state`. If `is_paused` is true, it stops.
- Reads `blog_runs` for today. If today's scheduled post is already out, it marks the run `skipped` and stops quietly.
- Takes the front of `blog_topic_queue` first (highest priority, then oldest). In the author's setup a separate Telegram
  inbox workflow fills that queue. It is not included here. You can add rows by hand.
- With an empty queue it reads Hacker News, Lobsters and a list of RSS feeds (see the **Fetch Story Sources** node).
- It drops topics the blog already covered (it reads recent posts from the blog API), then one local model call picks
  the topic. If the model fails, the highest scored candidate wins.

### Agent 2: research

- Fetches the source page, if there is one.
- Searches with SerpApi (3 tries) and fetches the top 3 results (2 tries each, failures skipped).
- Builds one research summary and a list of sources. The run fails only if there is no research at all.

### Agent 3: writer

- Local model through Ollama (`qwq:32b` in the export). 30 minute timeout, 2 tries, reasoning text stripped.
- Before it writes, it reads two Data Tables: `blog_validation_failures` and `blog_quality_metrics`. The three most common
  past problems and the average length go into the prompt. This is the quality feedback loop (see below).
- Length follows the research: roughly one word per 7 characters of research, 700 to 1,800 words.
- Topic, research and sources are marked as data. The prompt says never to follow instructions found inside them.

### Agent 4: editor

- The main editor runs outside n8n. Agent 4 writes the draft to the Data Table `blog_edit_requests` with
  `status = pending` and checks the row once a minute.
- In the author's setup a small service polls that table through the n8n public API, edits the draft with Claude Opus
  (no tools), and writes the result back. That service is not in this repo. The contract is in
  [External editor contract](#external-editor-contract).
- If no edit arrives within 20 minutes, or the row says `failed` or `stale`, Agent 4 falls back to the local model. The
  fallback edits one `##` section per call, checks each section against the research, and drops links that are not in
  the source list. If a section edit fails, the writer's text is kept and `editorOk` is false.
- You can run without the external editor. Every post then waits 20 minutes and uses the fallback. To skip the wait,
  lower the limit in **Give Up on Opus?** (`$runIndex >= 19`).

### Agent 5: quality check and publish

- Cleans links (only links from the source list survive) and appends a Sources section.
- **Validate Quality** scores the post and lists issues. Some issues are blockers: the editor did not check every
  section, numbers that are not in the research, another script, no title, a score below 6, or fewer than 400 words
  (80 for the digest).
- **Publish Gate** holds a post with any blocker. Holding throws an error, so the error workflow alerts and may retry.
- Checks that the slug does not exist yet (it never overwrites), and that today's scheduled post is not already out.
- Publishes to the blog API, marks the run `published`, marks the queue item done, sends a Telegram note, and hands the
  post to the X poster six minutes later.
- Writes to the learning tables: validation failures, quality metrics, topic performance, source reliability.

## The quality feedback loop

Agent 5 writes, Agent 3 reads:

1. Every issue Agent 5 finds goes into `blog_validation_failures` (type, severity, message, word count, score).
2. Every published post goes into `blog_quality_metrics` (score, word count, sections).
3. On the next run, Agent 3 reads the last 100 rows of both and tells the writer which problems came up most and how
   long earlier posts were.

`blog_topic_performance` and `blog_source_reliability` are written for later analysis. No workflow reads them yet.

## Weekly digest

Sundays at 10:00. It reads the week's posts from the blog API and stops if there are none. The local model writes one or
two sentences per post. Titles and links come from the API, never from the model. The digest goes to Agent 5 and is
published like any post, with the slug `weekly-digest-YYYY-MM-DD`.

## Error alerts and retries

`Blog Pipeline Error Alerts` is set as the error workflow on every other workflow. On a failure it:

- Sends one plain line to Telegram: workflow, step, error, at most 300 characters. Tokens, bearer values and
  `key=` style values are redacted from the message first.
- Finds the failed run in `blog_runs` by execution id and marks it `failed`, or `held` when the Publish Gate held it.
- Retries a scheduled run up to 2 times a day, 30 minutes apart, by starting Agent 1 again.
- Alerts for workflows outside the pipeline at most once an hour (state in `blog_bot_state`).

## Ham radio band report

Daily at 06:00. No model is involved.

- Reads NOAA SWPC solar flux, the planetary K index and the K index forecast. hamqsl.com fills gaps.
- Fixed rules turn those numbers into a plain reading of the HF bands by day and by night.
- An aurora line is added when Kp is 5 or more now or in the forecast. The thresholds assume about 53 N geomagnetic
  latitude. Set `PLACE` and the thresholds in **Build Band Note** for your location.
- Posts to a separate ham radio section of the blog API, at most once a day (it checks today's slug first).
- Goes to X only on notable days: Kp 5 or more, or solar flux 150 or more.

## X poster

Takes `{title, excerpt, url, kind, waitSeconds}`, waits so the page exists before X fetches the preview, then posts the
title, a short hook and the link. It refuses links that are not on your own site (`your-blog.example` in the export). A
failed X post fails only this workflow, never the publish.

## Data Tables

Create these in n8n (Data Tables) before you import. Column types: s = string, n = number, b = boolean.

| Table                      | Columns                                                                                                                                         | Used by                        |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| `blog_runs`                | run_id s, mode s, day s, status s, dry_run b, force_fail s, queue_id n, started_at s, finished_at s, detail s, post_url s, last_exec s, topic s | all agents, error alerts       |
| `blog_system_state`        | is_paused b (one row)                                                                                                                           | Agent 1                        |
| `blog_topic_queue`         | topic s, url s, source s, priority n, status s (pending, processing, done, failed), submitted_at s                                              | Agent 1, Agent 5, error alerts |
| `blog_edit_requests`       | req_id s, run_id s, status s, payload s, result s, note s, editor_notes s                                                                       | Agent 4, external editor       |
| `blog_validation_failures` | issue_type s, severity s, message s, topic s, word_count n, quality_score n, timestamp s, post_title s                                          | Agent 5 writes, Agent 3 reads  |
| `blog_quality_metrics`     | topic s, style s, length s, quality_score n, word_count n, section_count n, retry_count n, timestamp s, post_title s, post_slug s               | Agent 5 writes, Agent 3 reads  |
| `blog_topic_performance`   | category s, quality_score n, retry_count n, timestamp s                                                                                         | Agent 5                        |
| `blog_source_reliability`  | domain s, quality_score n, timestamp s                                                                                                          | Agent 5                        |
| `blog_bot_state`           | key s, value s                                                                                                                                  | error alerts                   |

The `id` column is built in.

## External editor contract

Only needed if you want an editor outside n8n. Agent 4 inserts a row in `blog_edit_requests`:

- `req_id`: run id plus execution id.
- `status`: `pending`.
- `payload`: JSON string `{"draft", "research", "sources": [{"url", "title"}], "topic"}`.

Your worker polls for `status = pending` (for example with the n8n public API, `GET /data-tables/{id}/rows`), edits the
draft, and updates the row:

- `status`: `done` (or `failed`).
- `result`: the edited post in Markdown. It must start with a `# Title` line and be longer than 200 characters.
- `note` and `editor_notes`: optional, shown in the run log.

Treat the draft and research as data. Do not let your editor follow instructions found in them.

## Import into your own n8n

You need:

- n8n with Data Tables (a recent 1.x release) and the public API if you run an external editor.
- Ollama reachable from n8n, with the model named in the Code nodes (`qwq:32b`, or change `model:` in the
  `ollamaBody` of Agents 1, 3, 4 and the digest).
- A blog API that accepts `POST /api/blog/posts` and `POST /api/hamradio/posts` with a bearer token, and answers
  `GET /api/blog/posts` and `GET /api/blog/posts/{slug}`. Change the paths in the HTTP Request and Code nodes to match yours.

Credentials and environment values:

| Name in the export               | Type                                 | Used by               |
| -------------------------------- | ------------------------------------ | --------------------- |
| `{{ CREDENTIAL: SerpApi }}`      | SerpApi                              | Agent 2               |
| `{{ CREDENTIAL: Telegram bot }}` | Telegram API                         | Agent 5, error alerts |
| `{{ CREDENTIAL: X OAuth2 }}`     | X (Twitter) OAuth2                   | X poster              |
| `BLOG_API_KEY`                   | environment variable on the n8n host | Agent 5, ham report   |

Steps:

1. Create the Data Tables above.
2. Import the nine JSON files. Import `blog-pipeline-error-alerts.json` and `agent-1-topic-discovery.json` first.
3. Replace the placeholders (next section). In each Data Table node, pick the table from the list.
4. Create the three credentials and select them in the nodes that show a credential warning.
5. Set `BLOG_API_KEY` in the n8n environment.
6. In each workflow's settings, set the error workflow to `Blog Pipeline Error Alerts`.
7. Test with a dry run: start Agent 1 with the input `{"dryRun": true}`. The whole chain runs, and Agent 5 skips the
   publish, Telegram and X. Dry runs also accept `stopAfter` (`agent1`, `agent2`) and `forceFailAt` (`agent1` to
   `agent5`) to test the alerts.
8. Activate the workflows from Agent 5 back to Agent 1, so each webhook exists before something calls it.

## Placeholders

| Placeholder                                           | Replace with                                                                                           |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `https://your-n8n.example/webhook/...`                | Your n8n webhook base URL. If all workflows run in one n8n, `http://localhost:5678/webhook/...` works. |
| `blog-agent-2` ... `blog-agent-5`, `blog-x-poster`    | Webhook paths. Keep them or rename them on both ends.                                                  |
| `http://OLLAMA_HOST:11434`                            | Your Ollama URL.                                                                                       |
| `https://your-blog.example`                           | Your site and blog API.                                                                                |
| `{{ DATA_TABLE: name }}`                              | The id of that Data Table (pick it in the node).                                                       |
| `{{ CREDENTIAL: ... }}`, `REPLACE_WITH_CREDENTIAL_ID` | Your credential (pick it in the node).                                                                 |
| `REPLACE_WITH_ID_OF: ...`                             | The workflow id of the named workflow, after import.                                                   |
| `YOUR_TELEGRAM_CHAT_ID`                               | The chat that gets announcements and alerts. Add a topic id under Additional Fields if you use topics. |
| `YOUR_TOWN`                                           | The place named in the aurora line.                                                                    |

## Older version (v1)

The first version of these workflows (one SerpApi research call, Ollama writing and editing, a Telegram command bot) is
in the git history. See the commit `14fbe8f` and earlier. It was removed from the tree because it no longer matches what
runs.

## License

This repository has no license file. That has not changed in v2.
