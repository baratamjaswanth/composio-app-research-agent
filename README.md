# 100-app research agent: which apps can be agent toolkits?

An agent researches 100 apps (auth, self-serve vs gated, API surface, MCP, buildability), a second stage
checks its evidence, a human checks a sample, and one static page presents the patterns.

## Run it

```bash
pip install -r requirements.txt
cp .env.example .env            # add ANTHROPIC_API_KEY

python agent/research.py --limit 3          # smoke test on 3 apps first
python agent/research.py                    # pass 1, all 100 (resumable)
python agent/evaluate.py --pass 1           # fetch cited pages, check quotes, LLM-judge, build human queue
python agent/research.py --pass 2           # re-research only flagged fields, with the reviewer's complaint
python agent/evaluate.py --pass 2

python analysis/sample.py                   # writes data/sample.csv (~30 apps, stratified + known-tricky)
# fill the `truth` and `truth_url` columns by hand from official docs (';' between multiple auth methods)
python analysis/score.py                    # field-level accuracy, pass 1 vs pass 2 -> data/accuracy.json
python analysis/patterns.py                 # -> site/data.js and site/data.json

cd site && python -m http.server 8000       # open http://localhost:8000
```

Optional: put `{"app": "...", "note": "..."}` entries in `data/notes.json` for apps that defeated the agent;
they show under "What went wrong" on the page. Deploy `site/` to GitHub Pages, Netlify or Vercel as a static folder.

## How it stays honest

- **Evidence rule.** Every non-unknown field needs a URL and a quote. No URL means the field is forced to `unknown` (`agent/schema.py`).
- **Quote check.** `evaluate.py` fetches each cited page and confirms the quote appears. Pages that cannot be fetched (JS-rendered, blocked) are listed as `fetch_failed` and go to the human queue; they are not counted as wrong.
- **Second model.** A different, cheaper model judges whether each quote supports the claimed value.
- **Pass 2 only touches flagged fields.** Unflagged apps are carried over unchanged, so the pass 1 to pass 2 change is attributable to the loop.
- **Scored against humans.** `score.py` compares both passes with your hand-checked values on the same sample.
- **Every number on the page is computed** by `analysis/patterns.py` from files on disk. With no results the page shows an empty state.

## Where a human is needed

1. Sample verification (the `truth` column).
2. Items in `data/human_queue_pass*.json`: fetch failures, unsupported quotes, agent failures.
3. Gated or JS-heavy pricing and signup pages. A browser-use or Playwright step is the natural next loop for these.

## Files

```
data/apps.json          the 100 apps (rebuild with data/build_apps.py)
data/results/pass1|2/   one JSON per app
agent/                  schema.py, research.py, evaluate.py
analysis/               sample.py, score.py, patterns.py
site/index.html         the case study (reads site/data.js)
```

## Swapping in Composio

`research.py` uses Anthropic's built-in web search tool. To use Composio's search/fetch tools instead, replace the
`tools=[...]` list and tool loop in `call()` with Composio tool calls (see docs.composio.dev for the current SDK for your framework).
Nothing else in the pipeline depends on the search backend.
