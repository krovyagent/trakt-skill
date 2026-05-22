# trakt-skill

An [Agent Skill](https://agentskills.io) that lets AI agents answer questions about your
[Trakt.tv](https://trakt.tv) activity — watch history, watchlist, trending titles, personal
recommendations, search, and lifetime stats — by invoking the
[`@raulanatol/trakt-cli`](https://www.npmjs.com/package/@raulanatol/trakt-cli) binary and
summarizing its JSON output in natural language.

Compatible with any agent that supports the open [Agent Skills](https://agentskills.io)
standard (Hermes Agent, Claude Code, Cursor, OpenCode, and others).

## What it does

Ask things like:

- "What did I watch this week?"
- "Is *Severance* trending?"
- "Recommend me a show."
- "Search for *Dune*."
- "Show me my Trakt stats."

The agent picks the right `trakt-cli` subcommand, parses the JSON, and replies with a
conversational summary instead of raw data.

## Requirements

- [Node.js](https://nodejs.org/) ≥ 18
- [`@raulanatol/trakt-cli`](https://www.npmjs.com/package/@raulanatol/trakt-cli) installed and on `PATH`:

  ```sh
  npm install -g @raulanatol/trakt-cli
  ```

- A Trakt API application — create one at <https://trakt.tv/oauth/applications> and export:

  ```sh
  export TRAKT_CLIENT_ID="..."
  export TRAKT_CLIENT_SECRET="..."
  ```

- Authenticate once via the OAuth device flow:

  ```sh
  trakt-cli login
  ```

## Installation

### Hermes Agent

From the Skills Hub:

```sh
hermes skills install raulanatol/skills/trakt
```

Or install locally:

```sh
mkdir -p ~/.hermes/skills/productivity/trakt
cp SKILL.md ~/.hermes/skills/productivity/trakt/
```

Then invoke it with `/trakt` in your Hermes session.

### Other agents

Drop `SKILL.md` (and any future supporting files) into the directory your agent reads skills
from. See the [Agent Skills client list](https://agentskills.io/clients) for the path used by
each tool.

## Available commands

The skill knows how to call:

| Command | Returns |
| --- | --- |
| `trakt-cli whoami` | Authenticated user info |
| `trakt-cli history` | Recent watch history |
| `trakt-cli watchlist` | Items on your watchlist |
| `trakt-cli trending` | Globally trending shows / movies |
| `trakt-cli recommendations` | Personal recommendations |
| `trakt-cli search <query>` | Search results |
| `trakt-cli stats` | Lifetime stats |

Write operations (mark as watched, rate, add to history) are not supported in `trakt-cli`
v1.0.0 and the skill will say so.

## Contributing

Issues and PRs welcome. If you find a question pattern the skill mishandles, open an issue
with the input and the expected behavior.

## License

MIT — see [LICENSE](./LICENSE).
