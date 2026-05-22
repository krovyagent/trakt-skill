---
name: trakt
description: Answer questions about the user's Trakt.tv activity — watch history, watchlist, personal recommendations, search, trending shows/movies, and lifetime stats. Use whenever the user asks what they've watched, what they want to watch, what's trending, or anything tied to their Trakt account. Invokes the `trakt-cli` binary and summarizes the JSON it returns.
version: 1.0.0
platforms: [macos, linux, windows]
metadata:
  hermes:
    category: productivity
    tags: [trakt, movies, tv-shows, watchlist, recommendations, entertainment]
    requires_toolsets: [terminal]
required_environment_variables:
  - name: TRAKT_CLIENT_ID
    prompt: Trakt API Client ID
    help: Create an application at https://trakt.tv/oauth/applications and copy the Client ID.
  - name: TRAKT_CLIENT_SECRET
    prompt: Trakt API Client Secret
    help: Same application as TRAKT_CLIENT_ID — copy the Client Secret.
---

# Trakt skill

Answer questions about the user's Trakt.tv activity by invoking the `trakt-cli` binary
(<https://www.npmjs.com/package/@raulanatol/trakt-cli>) via the terminal and summarizing the JSON
it returns. **Never dump raw JSON to the user.** Parse the JSON, extract the relevant fields,
and produce a natural-language answer.

## When to Use

Trigger on questions like:

- "What did I watch recently?"
- "What's on my watchlist?"
- "What shows are trending?"
- "Search for the movie X"
- "Recommend me something"
- "Show me my Trakt stats"
- Anything mentioning Trakt, viewing history, shows, or movies the user is tracking.

Do **not** use this skill for generic TV/movie info that doesn't involve the user's personal
data — answer from general knowledge instead. This skill is for *their* Trakt data.

## Prerequisites

1. `trakt-cli` installed and on `PATH`:

   ```sh
   npm install -g @raulanatol/trakt-cli
   ```

2. `TRAKT_CLIENT_ID` and `TRAKT_CLIENT_SECRET` exported (create the app at
   <https://trakt.tv/oauth/applications>).
3. The user is authenticated. Auth token lives at `~/.config/trakt-cli/auth.json` (mode 0600).
   The CLI manages it; never read or write that file directly.

If any of these are missing, follow the error handling rules below.

## CLI Contract

- **stdout** of every command is JSON. Parse it.
- **stderr** carries human prompts and `{"error": "...", "code": "..."}` on failure.
- **Exit code ≠ 0** signals failure — always check it before parsing stdout.
- `login` is interactive (OAuth device flow). **Never call it yourself.** Tell the user to run it.

## Procedure

### Available commands

| Command | What it returns |
| --- | --- |
| `trakt-cli whoami` | Authenticated user info (username, name, joined date) |
| `trakt-cli history [--type movies\|shows] [--limit N]` | Recent watch history. Default limit 10 if unspecified |
| `trakt-cli watchlist [--type movies\|shows] [--limit N]` | Items on the user's watchlist |
| `trakt-cli trending [--type movies\|shows] [--limit N]` | Globally trending titles on Trakt |
| `trakt-cli recommendations [--type movies\|shows] [--limit N]` | Personal recommendations |
| `trakt-cli search <query> [--type movie\|show]` | Search results, scored |
| `trakt-cli stats` | Lifetime stats (movies/shows/episodes watched, minutes, ratings, network) |
| `trakt-cli logout` | Clears stored token — only call if the user explicitly asks |

### Picking `--limit`

- Vague question ("what have I watched?") → `--limit 10`.
- Specific ("the last 3") → match the number.
- "Top N" → match N.

### Invocation

Run via the agent's terminal/shell tool. Examples:

```sh
trakt-cli history --limit 5
trakt-cli trending --type shows --limit 5
trakt-cli search "breaking bad" --type show
trakt-cli stats
```

### Summarizing the JSON

Don't enumerate every field. Pick what matters for the question.

**history** — each entry has `watched_at`, `type` (`episode` / `movie`), and either
`show.title` + `episode.title` / `episode.season` / `episode.number` or
`movie.title` + `movie.year`.

- Format episodes as `Breaking Bad S05E07 — "Say My Name"`.
- Format movies as `The Matrix (1999)`.
- Always include the watch date in a human-friendly form ("2 days ago", "yesterday", "May 12").

**watchlist / trending / recommendations** — each item has either `show` or `movie` with
`title`, `year`, `overview`, `ids.trakt`. Default format:

- **Title (year)** — one-line takeaway from `overview` (trim to ~120 chars).

If the user asks for details ("tell me about the first one"), expand the overview.

**search** — results are scored. Take the top match and ask whether the user meant it,
or list the top 3 with the type (show/movie) clearly marked.

**stats** — has `movies.watched`, `shows.watched`, `episodes.watched`, `episodes.minutes`
(total minutes), `ratings.total`, `network.followers` / `following`. Convert minutes to
days/hours for the user ("You've watched 124 days of content").

**whoami** — `username`, `name`, `joined_at`. One short sentence is enough.

## Pitfalls

Map non-zero exit codes to user-facing messages. Parse the JSON from stderr to read `code`.

| `code` field | Action |
| --- | --- |
| `ENOAUTH` | Tell the user: "You're not authenticated. Run `trakt-cli login` and ask me again." Do **not** try to log in for them. |
| Missing env vars (`TRAKT_CLIENT_ID` / `TRAKT_CLIENT_SECRET`) | Ask the user to export them and point to <https://trakt.tv/oauth/applications>. |
| Anything else | Surface the error message in plain language and stop. Don't retry blindly. |

If the shell reports `trakt-cli: command not found`:

> "You need to install the CLI: `npm install -g @raulanatol/trakt-cli`"

Other things to avoid:

- **Never echo raw JSON.** Summarize.
- **Never invent commands.** Write operations (mark as watched, add to history, rate) are
  not implemented as of `trakt-cli` v1.0.0 — say so if the user asks.
- **Don't call `trakt-cli login`** — it's interactive. Tell the user to run it.
- **Don't read `~/.config/trakt-cli/auth.json`** directly.

## Verification

Quick sanity check after installing/configuring:

```sh
trakt-cli whoami
```

If it returns a JSON object with `username`, the skill is ready. Any non-zero exit means
auth or env vars are missing — handle per the pitfalls table.

## Style

- Conversational; only use bullet lists when the data itself is a list (history, search, watchlist).
- Keep show/movie titles in their original language.
- Respond in English.
- Don't apologize for not knowing — if a command fails, state what failed and what the user
  can do next.

## Example Interactions

**User:** "What did I watch this week?"
→ Run `trakt-cli history --limit 20`. Filter entries with `watched_at` in the last 7 days.
Summarize as a short list: "This week you watched X episodes and Y movies: …".

**User:** "Recommend me a show."
→ Run `trakt-cli recommendations --type shows --limit 5`. Pick the top 1–2 and explain why
they fit (based on the `overview`, don't invent reasons).

**User:** "Is Severance trending?"
→ Run `trakt-cli trending --type shows --limit 20`. Look for "Severance" in the results.
If present, mention its position; if not, say so clearly.

**User:** "Search for Dune."
→ Run `trakt-cli search "Dune"`. List the top 2–3 results with type (movie/show) and year.
