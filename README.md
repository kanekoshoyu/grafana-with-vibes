# grafana-with-vibes

Pull Grafana dashboards as local JSON, edit them with Claude Code (or manually), and push them back — no browser required.

## Setup

**1. Get a Grafana service account token**

In Grafana: Administration → Service accounts → New service account (Editor role) → Add token

**2. Clone and initialize**

```bash
git clone https://github.com/your-username/grafana-with-vibes
cd grafana-with-vibes
./grafana_with_vibes init --url http://your-grafana:3000 --token glsa_xxx
```

This saves credentials to `.env` (git-ignored) and verifies connectivity.

## Usage

```bash
# List all dashboards with their UIDs
./grafana_with_vibes list

# Pull a single dashboard (use UID from list)
./grafana_with_vibes pull --uid abc123def

# Pull all dashboards
./grafana_with_vibes pull --all

# Push a dashboard back to Grafana
./grafana_with_vibes push dashboards/my-dashboard__abc123def.json
```

## Editing with Claude Code

Open this repo in Claude Code. Once your dashboard is pulled, ask Claude to load and edit it:

> "Load the Squeeze Analysis dashboard and add a panel showing open interest over time"

Claude will use `./grafana_with_vibes list` and `pull` to fetch the dashboard, read the JSON, make edits, and push it back. The `CLAUDE.md` in this repo gives Claude the context it needs to do this correctly.

## File layout

```
grafana-with-vibes/
├── grafana_with_vibes   # CLI tool
├── .env                 # Grafana URL + token (git-ignored)
├── .env.example         # Template
├── dashboards/          # Downloaded dashboard JSON (git-ignored by default)
├── CLAUDE.md            # Instructions for Claude Code
└── README.md
```

## Dashboard file format

Each file in `dashboards/` looks like:

```json
{
  "dashboard": { ... },    // the full Grafana dashboard spec — edit this
  "folderUid": "abc",      // preserved on push
  "folderTitle": "General"
}
```

Edit only the `dashboard` object. The CLI wraps it correctly for the Grafana API on push.

File names follow the pattern `{slug}__{uid}.json`, e.g. `squeeze-analysis__b33579fa.json`.

## Requirements

- `bash`, `curl`, `python3` — standard on macOS and Linux
- Grafana with a service account token (Editor role or higher)

## Version controlling dashboards

`dashboards/` is git-ignored by default. To track your dashboards in git, remove it from `.gitignore`. Be careful not to commit tokens or sensitive query data.
