# 07 · Operations, server, testing and lessons learned

## 1. Backend server

FastAPI app served by Uvicorn on `127.0.0.1:<port>` (default 8765), the same process serving the built frontend (`frontend/dist`: `/assets/*` and an `index.html` fallback for any non-API path). On start-up (lifespan): create/migrate tables, then **start the background sync** in a thread guarded by a lock. Bind to localhost only.

**API conventions**

- All data endpoints are `GET /api/...` and take `period` ∈ `all | 2y | 1y | 6m | 3m | 1m` (default `6m`); any other value → **HTTP 400**. Unknown ride → **404** with a message.
- Endpoints: `meta, sync (GET/POST), overview, power, radar, ride/{id} (0 = latest), quadrant, decoupling, compare, recovery, correlations, climbs, equipment, nobike (rides to match), coach, quality, activities, bikes (CRUD), activities/{id}/bike, components (CRUD), pauses (GET/POST/DELETE)`.
- **Every endpoint must return JSON.** Because a static-file catch-all route serves `index.html` for unknown paths, an endpoint that was deleted by mistake silently returns HTML and the page hangs on "Loading…". After **every** backend edit run a smoke test that calls every endpoint and parses the response as JSON.
- Pydantic models for bodies; return `{ "ok": true }` for mutations; send the server's message in `detail` for 400s (the UI shows it).

## 2. The `cadenzAI.sh` script

Commands: `start | stop | restart | status | logs [N] | run | ai start|stop|status`. Behaviour to reproduce:

- `start`: if already running say so and exit 0; otherwise refuse politely if **another program** uses the port (never touch it); launch Uvicorn **fully detached** and wait (≤ 20 s) for `GET /api/meta` = 200; write the PID file `data/server[-<port>].pid`; print the URL.
- `stop`: find the PID (PID file, else the listener of the port) and verify with `/proc/<pid>/cmdline` that it is **uvicorn `app:app` started on this very port** before sending `SIGTERM`; escalate to `SIGKILL` after 10 s; warn if a Garmin update is running ("it resumes on the next start"). Exit 0 if not running.
- `restart` = stop + start. `status`: PID, uptime, URL, HTTP code, the sync state and the log path; **exit code 0 if running, 3 if not**. `logs`: no argument → follow the log; a number → last N lines (validate it). `run`: foreground.
- With the optional local AI (`10`), `start` also launches Ollama (loopback only) if installed and not running, and `stop` stops it only if the script started it. `status` prints a "local model" line (not installed / off / on with the installed models); `ai start|stop|status` manages only the model server; `CADENZAI_AI=0` prevents it from ever being started.
- Host/port from environment variables (`<APP>_HOST`, `<APP>_PORT`); the PID file is **per port**.

Lessons (each one cost a failed run in the reference build):

1. **Never `pkill -f '<pattern>'`** when the pattern also appears in your own command line: it kills your shell. Kill by PID.
2. **Verify that the PID is yours and on this port** before killing: a PID file from another port once made `stop` kill the wrong instance.
3. **Detach properly.** Launch with the redirections applied to the subshell *before* forking and `exec setsid nohup uvicorn …`; otherwise the child inherits the caller's stdout and `./cadenzAI.sh restart | tail` never terminates (any caller that waits for end-of-output hangs).
4. Background-task "completed" notices only mean the *launcher* finished: check the log and the DB.

### 2.1 Windows

On Windows prefer **WSL 2** (Ubuntu): everything above works unchanged inside it. For a native Windows build, ask for a `cadenzAI.ps1` (PowerShell) with the same commands, exit codes, per-port PID file and URL, replacing the Linux-only parts:

| Linux | Windows (PowerShell) |
|---|---|
| `setsid nohup uvicorn … &` | `Start-Process -WindowStyle Hidden -RedirectStandardOutput/-RedirectStandardError <log> -PassThru` (no console window, output to the log) |
| listener of the port (`ss`/`lsof`) | `Get-NetTCPConnection -LocalPort <port> -State Listen` → `OwningProcess` |
| `/proc/<pid>/cmdline` | `(Get-CimInstance Win32_Process -Filter "ProcessId=<pid>").CommandLine` must contain `app:app` and `--port <port>` |
| `SIGTERM`, then `SIGKILL` | `Stop-Process -Id <pid>`, then `-Force` after 10 s |
| `tail -f` / `tail -n N` | `Get-Content <log> -Wait` / `-Tail N` |

Windows lessons: the paths of the virtual environment are `.venv\Scripts\python.exe`; allow `.ps1` scripts for the current user with `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`; never kill by process name (`Stop-Process -Name python`), only by verified PID; open files cannot be deleted on Windows, so stop the server before touching `data/`.

## 3. Build and run

```bash
python3 -m venv .venv && .venv/bin/pip install fastapi uvicorn httpx fitparse numpy scipy pydantic
cd frontend && npm install && npm run build      # produces frontend/dist
./cadenzAI.sh start                                  # http://127.0.0.1:8765
```

On Windows (PowerShell): `py -3.12 -m venv .venv; .venv\Scripts\pip install fastapi uvicorn httpx fitparse numpy scipy pydantic`, then the same `npm` commands and `.\cadenzAI.ps1 start`.

Rebuild the frontend after every UI change (`npm run build`); restart the server after backend changes.

## 4. Verification protocol (run at the end of every phase)

1. **Data checks** listed in `04` for the metric you just built (print the numbers, compare with Garmin where a reference exists).
2. **Endpoint smoke test**: every endpoint × a couple of periods returns JSON; invalid period → 400; unknown id → 404.
3. **Full-page scan in a real browser** (Chrome DevTools MCP or Playwright): for each menu entry and for at least two periods, wait until "Loading" disappears, then assert there is no console error and no `NaN`, `undefined`, `Infinity` or `[object` in the visible text; list pages with and without the period filter.
4. **Interaction checks**: open/close modals (Esc, outside click, ✕, focus return), accordion toggles and persistence after reload, column sorting (incl. N/D last), switches (theme, views, x-axis), links from tables to the ride page, previous/next ride.
5. **Look at it.** Take screenshots of the page you changed in dark **and** light mode and review layout, overlaps, clipped content, label collisions. Charts that animate on entry give half-drawn screenshots: that is one reason to disable animations.
6. **Idempotence test** of the sync (`03` §3) and a **data-integrity count** (rides, FIT files, rows per derived table) before/after any migration.
7. **Report honestly:** what you verified, what you could not, what failed.

## 5. Project memory (notes)

Keep a short file (for Claude Code: its memory directory, or `NOTES.md`) with the **user's decisions that the code cannot show**: weight and max HR chosen, bike rules, thresholds, naming, what was deliberately not built, and the user's working preferences (e.g. "do not stop after saying you will proceed", "wait for all answers to a list of questions"). Update it whenever a decision changes (menu names, retention, defaults).

## 6. Lessons learned (checklist of mistakes to avoid)

| Area | Lesson |
|---|---|
| Garmin API | `endDate` mandatory; `activityType` is an object; gear endpoints 404; handle 429 with waits; keep long jobs resumable |
| Time | FIT times are UTC; attach `tzinfo` before `.timestamp()`; match export files by nearest start time |
| FIT | `left_right_balance` has a flag bit; drop `unknown*` fields; keep original files |
| Data fields | export `restingHeartRate` is a constant, the daily value is `currentDayRestingHeartRate`; verify every export field against the API |
| Cleaning | the first spike filter was too aggressive (it removed real standing starts), the second too weak (missed pushes at low speed): validate on the largest removed groups and on the best-1 s/5 s curve |
| Metrics | split decoupling on **moving samples**, not wall-clock time; first-42-day CTL warm-up; moving hours (not elapsed) for suspension |
| Thresholds | do not publish a wear threshold without a source; unsupported 50 h caused a false alarm |
| DB | tables created by an older version need `ALTER … ADD COLUMN` migrations; never delete by age |
| Backend | a deleted endpoint returns the SPA's HTML → frontend "Loading…" forever: smoke-test all endpoints; do not name a module like a stdlib module |
| Frontend | a modal's focus effect must not depend on an inline `onClose` (it steals the focus on every keystroke: keep it in a ref); lists cached in a context must be reloaded when a page that depends on other edits opens; hooks before early returns; escape `<` in JSX text; no fixed-height cards around content of unknown size; no inner scrollbars; synced charts via `syncId` |
| Ops | no `pkill -f`; verify PID ownership/port; detach fully; read logs, not launcher exit codes |
| Collaboration | the user may be using the app while you test: use a separate browser window, never confirm a destructive dialog you did not open, and restore any state you changed during tests (e.g. delete test components) |
| Honesty | say when a figure is a proxy, when a check could not be run, and when your first answer was wrong |
