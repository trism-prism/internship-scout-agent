# internship-scout-agent

Single-user weekly agent: pulls internship postings from a public feed, scores them against a resume with Claude tool calling and emails a digest; FastAPI on Vercel serves the API and public page.

## Commands
- Setup: `pip install -r requirements-dev.txt`
- Run locally: `cp .env.example .env` (set `API_TOKEN`), then `docker compose up --build` (Postgres + app on port 8000)
- Scoring job: `docker compose exec app python -m app.run` (real Claude API spend)
- Check: `ruff check .` and `TEST_DATABASE_URL=postgresql://postgres:postgres@localhost:5432/postgres pytest --cov=app --cov-fail-under=80`
  - DB tests skip without `TEST_DATABASE_URL`; CI provides a Postgres service

## Map
| Path | What lives there |
|---|---|
| `app/main.py` | FastAPI API + public page (Vercel entry point) |
| `app/run.py` | weekly job: fetch feed, score, send digest (`.github/workflows/weekly.yml`) |
| `app/agent.py` | manual tool-calling loop over the Messages API with a turn limit |
| `app/feed.py`, `app/digest.py` | feed fetch/upsert; HTML digest sent via Resend |
| `app/db.py`, `app/schema.sql` | psycopg helpers; schema applied at startup |
| `app/demo_data.py` | fictional data behind `/demo` |
| `scripts/record_demo.py` | records `docs/demo.gif` |

## Non-obvious facts
- Posting text is untrusted. The model's only tools are `get_posting_details` (by DB id, never a URL) and `record_match`. Description fetches allow public IPs over https only, with redirect and size limits; output is HTML-escaped. Code sends the email, not the model.
- Plain SQL with psycopg; no ORM or migration tool.
- Tests never call the real API: the anthropic SDK gets a mocked httpx transport.
- Real resumes are never committed (`resume*` is gitignored). Demo and tests use a fictional sample.
- Explanations quote resume content, so `/matches` endpoints need the bearer token; the public page shows company, title and score only. Empty `API_TOKEN` makes protected endpoints return 503.
- Scoring runs in GitHub Actions, not on Vercel: a run exceeds the Hobby plan's 300 s function limit. `weekly.yml` runs Mondays 13:00 UTC only when the repo variable `SCOUT_ENABLED` is `true`. A cancelled run stays "running" until the 4 h stale check clears it.
- Vercel installs from `pyproject.toml`, not `requirements.txt`; `tests/test_dependencies.py` requires both lists to match, so edit both.
- `SCOUT_MODEL` defaults to `claude-haiku-5-5`.
- `scripts/record_demo.py` needs playwright and pillow installed by hand (not in requirements). Without `--live` it seeds fictional data and refuses non-local databases.
- Single-user on purpose (each user is real API spend). Read the README section "Scaling to multiple users" before adding multi-user features.
