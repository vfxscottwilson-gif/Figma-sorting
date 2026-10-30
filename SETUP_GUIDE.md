# Sheet Organizer: Setup Guide and Roadmap

## 0. What this is, and what it is not (read first)

| You asked for | What really happens |
|---|---|
| Paste data randomly, the robot sorts it daily | **Yes.** Paste into the `Inbox` tab. Once a day (or when you press Run) the robot files every note into the right tab. |
| Tabs created automatically | **Yes.** You give it the sheet and the name of your paste tab. It builds the rest. |
| Month-end view: problems, solutions, progress, due dates, forgotten items | **Yes.** `Monthly Summary` (numbers + a written report) and `Attention Needed` (overdue / blocked / forgotten). |
| Figma board always sorted and arranged | **Partly, and this is a hard limit of Figma, not of the code.** Figma's web API can *read* a board and *post comments*. It cannot move, create or rearrange things on the canvas (that needs a Figma plugin running inside Figma). So: the robot **reads** your Figma / FigJam sticky notes and text into the sheet, and the **sheet** is the sorted view. Optionally it posts a comment on the board with the sheet link. |
| Gmail sorting | **Not included.** Gmail needs a separate OAuth login flow. Say so if you want it as phase 2. |
| No model names | **Done.** You only add API keys. Note: every AI provider's API technically needs a model name inside the request. The script gets it itself by asking each provider "which models can this key use today?", so you never type one and retired names cannot break it. |

## 1. What you need to provide (the full list)

1. **A Google Sheet** (one file). Paste tab name: `Inbox`. If you want another name, set the variable `INBOX_TAB`.
2. **A Google service account** (the robot's login):
   - Google Cloud Console, create a project, **enable "Google Sheets API"**.
   - IAM and Admin, Service Accounts, create one, Keys, Add key, JSON. A `.json` file downloads.
   - Open the Sheet, Share, paste the service account's email (`...@...iam.gserviceaccount.com`) as **Editor**.
3. **At least one AI key** (more = safer): Gemini (Google AI Studio), Groq, OpenRouter. All have free tiers. Several keys per provider: separate them with commas.
4. **A GitHub repository** (private recommended) containing these files.
5. *(Optional)* **Figma personal access token** with scopes `file_content:read` (and `file_comments:write` only if you want comments), plus the board link.

## 2. Files to upload to GitHub (exact paths)

```
organizer.py
requirements.txt
.github/workflows/organize.yml
```

## 3. GitHub setup (10 minutes)

Repository, Settings, Secrets and variables, Actions.

**Secrets**
| Name | Value |
|---|---|
| `SHEET_ID` | the long id in the sheet URL between `/d/` and `/edit` |
| `GOOGLE_SA` | the **entire content** of the service-account JSON file |
| `GEMINI_API_KEY` | your key (optional, but add at least one AI key) |
| `GROQ_API_KEY` | your key (optional) |
| `OPENROUTER_API_KEY` | your key (optional) |
| `FIGMA_TOKEN` | optional |

**Variables** (all optional): `FIGMA_FILE` (board link or key), `FIGMA_COMMENT` (`yes`), `INBOX_TAB`, `TIMEZONE` (default `Asia/Karachi`), `RUN_LIMIT` (default 150), `FIGMA_PAGE` (only read one page of the board), `FIGMA_NODE_TYPES` (default `STICKY,TEXT,SHAPE_WITH_TEXT`).

Then: **Actions tab, "Organize my sheet", Run workflow**, set `dry_run` = `yes` for the first test (writes nothing, shows counts). Run again with `no`.

## 4. How to use it every day

Paste anything in `Inbox`, one note per row (or paste a block; one note can contain several items and the AI splits them). Examples:

```
how to fix the login redirect issue
Fix homepage banner by Friday, urgent, Ali
finished the new logo animation, client approved
https://example.com/free-icons   nice icon pack
login redirect fixed by clearing the cache
```
The last line **updates** the existing problem row (solution added, status Solved). It does not create a duplicate.

Never edit or delete the Inbox for the robot's sake: it remembers every note by fingerprint in `_Ledger`. You may clear the Inbox whenever you like.

## 5. Tabs the robot creates

| Tab | Purpose |
|---|---|
| `Attention Needed` | Overdue, blocked, due in 7 days, open 14+ days with no due date ("forgotten?"), problems with no solution after 7 days. Rebuilt every run. |
| `Monthly Summary` | Per month: problems, solved, open, tasks, done, in progress, blocked, overdue, completion %, achievements + a short written manager's report (AI, from your real numbers only). |
| `Tasks & Progress` | ID, date, month, task, status, priority, owner, due date, category, notes, source |
| `Problems & Solutions` | ID, date, month, category, problem, solution, status, priority, owner, due date, tags, source, notes |
| `Achievements` | the month's wins and records |
| `Ideas & Notes`, `Resources & Links` | everything else |
| `Review Needed` | notes the AI could not understand. Rewrite and paste again. |
| `_Ledger`, `_Log` | the robot's memory and run history. Do not delete `_Ledger` (it would re-file everything; duplicates are still caught by title matching). |

You may edit **Status / Priority / Owner / Due Date** by hand in the filing tabs (dropdowns are provided). The robot respects your edits and only updates a row when a new note about the same item arrives. Do not edit `Attention Needed` or `Monthly Summary`: they are rebuilt each run.

## 6. Problems already prevented (and how)

| Risk | Protection |
|---|---|
| Model name wrong / retired / not offered | Keys only. Models are discovered live per provider; failed ones are skipped automatically; a built-in fallback list exists. |
| One provider out of quota | Automatic failover Gemini, Groq, OpenRouter, across all keys. Finished notes are saved before the next batch; unfinished ones wait for the next run. |
| AI invents data | Output is validated in code: type, status, priority and dates are normalised; unusable answers go to `Review Needed`; nothing is invented into blank fields. |
| Same note filed twice | Fingerprint ledger + similar-title matching (84% likeness). |
| Prompt injection from pasted text | Notes are marked as untrusted data in the prompt, and the AI can only output fields that are validated. |
| Two runs at once | `concurrency` lock in the workflow. |
| Sheet rate limits / outages | Retries with back-off; writes are batched. |
| Someone renames or reorders columns | Columns are found by name; missing ones are added at the right. |
| Spreadsheet formula injection | Everything is written as plain text. |
| Secrets leaking in public logs | The script never prints notes, names or keys. |
| Wrong dates | `dd/mm/yyyy` is read day-first; "Friday" etc. resolved against today's date in your time zone. |
| Daily schedule silently stopping | GitHub **pauses scheduled workflows after 60 days with no repository activity**. Press "Run workflow" or push any small change occasionally. A red run means no AI worked; check your keys. |

## 7. Honest limits

- Free AI tiers have daily caps. At about 150 notes per run it normally fits; extra notes wait for the next day. Add a second provider key to double the room.
- Matching "same item" is by title likeness, so two very differently worded notes about one issue can become two rows. Merge by hand if that happens.
- The Figma reader depends on what Figma returns for your file. Run once with `dry_run=yes` and read the log line "Figma: N items read". If N is 0, check the token scope, the file link and `FIGMA_PAGE`.
- Design (non-FigJam) files hold lots of UI text; use `FIGMA_PAGE` or keep a dedicated FigJam board for notes.
- GitHub cron can start a few minutes late. The robot runs once a day, not live.

## 8. If a run goes red

| Message | Fix |
|---|---|
| `GOOGLE_SA ... not the full JSON` | Paste the whole file content, including the braces. |
| `Cannot open the sheet` | Wrong `SHEET_ID`, Sheets API not enabled, or sheet not shared with the service-account email as Editor. |
| `No AI key found` | Add one of the three API key secrets. |
| `all AI providers failed` | Keys invalid or free quota used up. Add another key or wait a day. |
| `INBOX_TAB ... not found` | Fix the name (it must match exactly) or delete the variable. |
