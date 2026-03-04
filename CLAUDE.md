# grafana-with-vibes — Claude Code Instructions

This project lets you pull Grafana dashboards as local JSON, edit them, and push them back. The CLI is `./grafana_with_vibes`.

## Workflow

```bash
./grafana_with_vibes list                        # find dashboard UIDs
./grafana_with_vibes pull --uid <uid>            # download to dashboards/
./grafana_with_vibes push dashboards/<file>.json # push edits back
```

## Loading a dashboard

1. Run `list` to find the UID of the target dashboard.
2. Run `pull --uid <uid>` to save it to `dashboards/{slug}__{uid}.json`.
3. Read the file. **Dashboard JSON files can exceed 256KB** — use `offset` and `limit` with the Read tool, or use Grep to locate specific panels/queries.

```bash
# Find a panel by title
grep -n "Panel Title" dashboards/my-dashboard__abc123.json

# Or use the Grep tool with a pattern like:
# pattern: "\"title\":\\s*\"Open Interest\"", path: "dashboards/"
```

## File structure

Each file in `dashboards/` wraps the Grafana API response:

```json
{
  "dashboard": { ... },     // <-- EDIT ONLY THIS OBJECT
  "folderUid": "abc",       // preserved on push, do not change
  "folderTitle": "General"  // informational only
}
```

**Only edit inside `"dashboard"`.** The push command wraps it correctly for the API.

## Dashboard JSON structure

Key fields inside the `dashboard` object:

- `panels[]` — array of panel objects. Each panel has:
  - `title` — panel name
  - `type` — visualization type (`timeseries`, `table`, `stat`, `gauge`, etc.)
  - `targets[]` — array of queries (datasource-specific)
  - `fieldConfig` — display/unit/threshold settings
  - `gridPos` — position `{x, y, w, h}` on the grid (24 columns wide)
- `templating.list[]` — dashboard variables (e.g. `${TOKEN}`)
- `time` — default time range
- `annotations.list[]` — annotation queries

## Editing tips

- **Add a panel**: append to `panels[]`. Copy an existing panel as a template, change `id` to a new unique integer, update `gridPos`.
- **Edit a query**: find the panel by title, then edit `targets[].rawSql` (for QuestDB/SQL datasources).
- **Change layout**: update `gridPos.x`, `gridPos.y`, `gridPos.w`, `gridPos.h`. Grid is 24 units wide.
- **Add a variable**: append to `templating.list[]`.
- **Panel IDs must be unique** within the dashboard.

## QuestDB SQL queries (this project's datasource)

Time filtering pattern used throughout:
```sql
WHERE ts >= timestamp '1970-01-01T00:00:00.000Z' + $__from * 1000L
  AND ts <  timestamp '1970-01-01T00:00:00.000Z' + $__to   * 1000L
```

Token variable: `'${TOKEN}'` — used in `WHERE coin = '${TOKEN}'`.

Datasource uid: `bf3a9i7tmeio0d` (type: `questdb-questdb-datasource`).

## Testing queries before pushing

**Always test SQL changes against QuestDB before pushing.** Use the HTTP API:

```python
import urllib.parse, urllib.request, json
from datetime import datetime, timezone

# Pick a range with known data (liq data: 2025-07-28 to 2026-01-31)
from_ms = int(datetime(2025, 8, 1, tzinfo=timezone.utc).timestamp() * 1000)
to_ms   = int(datetime(2025, 8, 10, tzinfo=timezone.utc).timestamp() * 1000)

sql = "...your query with $__from/$__to replaced by {f}/{t}...".format(
    f=from_ms*1000, t=to_ms*1000  # QuestDB uses microseconds
)
url = "http://questdb.bounteer.com:9000/exec?query=" + urllib.parse.quote(sql)
with urllib.request.urlopen(url, timeout=120) as r:
    d = json.loads(r.read())
    if 'error' in d:
        print("ERROR:", d['error'])
    else:
        print(f"OK — {d['count']} rows:", d['dataset'][:5])
```

Key dates with liq event data: 2025-08-02 (VINE/PENGU stress events), 2025-08-09 (MOODENG). Use `volume_threshold=1` in tests to bypass volume filter.

## Pushing changes

```bash
./grafana_with_vibes push dashboards/my-dashboard__abc123.json
```

On success it prints the Grafana URL. On error it prints the API response — check for JSON parse errors or invalid panel config.

## Requirements

- `.env` must exist with `GRAFANA_URL` and `GRAFANA_TOKEN`. Run `./grafana_with_vibes init` if missing.
- Needs `bash`, `curl`, `python3`.
