# Talent 360i — Auto Assessment Generation with SME in the Loop

AI agents turn NIQ FinOps role/skill data into skill- and level-matched assessments. People
make every decision: an **SME approves each AI-generated question** before anyone sees it, and a
**manager confirms the AI's evidence evaluation** before any training plan or level change.

All employee data is the synthetic Hackfest dataset in `data/`.

---

## Quick start (about 5 minutes)

**You need:** Python **3.12**, internet access on the first run (packages, plus a small
embedding model), and an LLM key: Groq for a personal machine, or the NIQ CIS / Azure AI gateway.

**macOS / Linux**
```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python start.py --setup        # creates .env files, generates secrets, builds the database
```

**Windows (PowerShell)**
```powershell
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python start.py --setup
```

Next, open `.env` in the project root and add **one** LLM key:

```ini
GROQ_API_KEY=gsk_...                 # personal machine (default LLM_PROVIDER=groq)
# or, on the NIQ network / VPN:
LLM_PROVIDER=cis
CIS_API_KEY=Bearer ...
```

Then start both servers:

```bash
python start.py
```

Open **http://127.0.0.1:8010/login**. Press `Ctrl+C` to stop both servers.

| Log in as | Password |
|---|---|
| `employee`, `manager`, `sme`, `scheduler`, `leader`, `admin` | value of `TALENT360_DEMO_PASSWORD` in `.env` |
| `syn-u001` … `syn-u036` (synthetic dataset users) | value of `TALENT360_SYNTHETIC_PASSWORD` in `.env` |

`start.py --setup` generates both passwords the first time, writes them into `.env`, and never
prints them. To choose your own, edit `.env` and run `python start.py --reseed`.

---

## Demo walkthrough (the full human-in-the-loop flow)

| # | Log in as | Do this | What it shows |
|---|---|---|---|
| 1 | `employee` | Dashboard → upload `data/sample_resumes/finops_ap_p2p_senior_associate.pdf` | Agent 1 matches the resume to a role. The employee only uploads; they don't choose the number of questions |
| 2 | `sme` | **AI Generation** → role, level, count → Review Queue → **Approve** | Agent 2 writes and self-checks each question at the level the SME chose; approved questions join the reusable **Question Bank** |
| 2b | `sme` | **Assign Assessments** → **Set up assessment** → adjust the proposed questions → **Deploy** | The selection agent proposes bank questions (priority skills, level, least-used); the SME decides. The bank starts with the dataset's SME-approved questions for all 9 roles. Anyone not yet evaluated in Talent360i for the role they hold (dataset levels alone don't count) gets an assessment built from approved bank questions automatically |
| 3 | `employee` | Dashboard → **Start assessment** → answer → submit | Agent 3 scores it; the score is visible only to the SME and manager |
| 4 | `employee` | **Continue simulation** → work through each scenario and answer the AI assessor's follow-ups | Agent 4 runs a workplace simulation, probing up to 3 times per skill for missing evidence, then builds an evidence report from the whole transcript: an evaluation only, never a decision |
| 5 | `manager` | Validations → open the cycle → review the per-skill outcome → **Approve & certify ticked levels** | The recommendation (strong / low) is only a proposal. Ticked skills become *validated* at the level tested; Agent 5 outlines skill-up sessions towards the next role's targets and books them on the calendar |
| 5b | `manager` | …or **Reject & assign skill-gap training** / **Send back to retake** | Reject records the levels shown as confirmed gaps and books an Agent 5 skill-gap outline; when the manager marks every session delivered, a re-assessment on those skills is auto-assigned. Send back assigns a fresh retake (e.g. suspected AI use) |
| 6 | `employee` | **My Training Plan**, **My Skills & TNI** | The training outline from the manager's decision (objectives, activities, practice task, how it's checked, booked dates), validated levels, and the career path to the next role |
| 7 | `leader` | **Organisation Report**, **Manager Calibration** → Review evidence → Concur / Send for re-review | Live org reporting with CSV export; decisions that run against the evidence are flagged, and a re-review returns the decision to the manager (the leader never overrides it) |
| 7b | `manager` | **Assign Training** → enrol a report in a recommended course, then mark it completed | Enrolments appear on the employee's training plan and calendar |
| 8 | `syn-u001` … / `manager` | **My Skills & TNI** / **Learning & TNI** | Skill heatmap, projected learning curve, and recommended learning with gap coverage from the synthetic TNI baseline |
| 9 | `scheduler` | **Schedule & Calendar** → book, or auto-schedule TNI training | Month calendar of every booked assessment and training session; employees see their bookings on their training plan |
| 10 | `manager` → `leader` | **Promotion Readiness** → open a person → **Assign promotion assessment** / **Send profile for promotion review**; then leader **Promotions** → Approve / Decline | Readiness against the next role up the tower (B3 → B4 → P3 → P4). The leader sees readiness, certified levels, every assessment with AI-use flags and the manager's case. An approval is recorded for HR; job records aren't changed |

Generating questions takes about 15–40 s. In the workplace simulation each assessor follow-up takes a few seconds, and the evidence report and plan take 20–40 s each.

---

## Architecture

```
Browser ──session + CSRF──▶ Portal  (Talent360i_phase11C_fixed_remote_hybrid/app.py, port 8010)
                             │  RBAC for 6 roles, workflow state in SQLite, AI-notice banners
                             │  Bearer token (shared secret in both .env files)
                             ▼
                        Pipeline API  (app/main.py, port 8000, JSON only)
                             │  upload + input validation, rate limits, audit log
                             ▼
     agents/  A1 match (Chroma) → A2 generate + self-check → A3 score → A4 simulation + evidence → A5 plan + training outline
                             │  single LLM choke point: agents/llm_client.py
                             ▼
                        Groq (dev) or CIS / Azure AI Inference (approved provider)
```

Both processes listen on `127.0.0.1` only. `start.py` launches both and stops both.

| Path | What |
|---|---|
| `start.py` | One-command setup and launcher |
| `app/main.py` | Pipeline API (FastAPI) |
| `agents/` | The five agents, `llm_client.py`, shared `security.py` controls |
| `Talent360i_phase11C_fixed_remote_hybrid/` | Portal: routes (`app.py`), templates, services, workflow DB layer |
| `scripts/` | Database build and seed scripts (run by `start.py`) |
| `data/` | Synthetic Hackfest dataset + sample resumes |
| `tests/` | Security-control and workflow tests |
| `SECURITY.md` | Guideline → control → test map, threat model, known limitations |

---

## Configuration

Everything lives in `.env` (project root) and `Talent360i_phase11C_fixed_remote_hybrid/.env`. Both
files are git-ignored; see the `.env.example` next to each one.

| Variable | Default | Notes |
|---|---|---|
| `LLM_PROVIDER` | `groq` | `groq` or `cis`. Groq is refused when `TALENT360_ENV=production` |
| `GROQ_API_KEY` / `CIS_API_KEY` | – | The only value you must set yourself |
| `TALENT360_BACKEND_TOKEN` | generated | Must match in both files; `start.py` keeps them in sync |
| `TALENT360_DEBUG` | `true` | Set `false` for any shared deployment (turns on HSTS, Secure cookies, required secrets) |
| `TALENT360_API_PORT` / `TALENT360_PORTAL_PORT` | `8000` / `8010` | Set as environment variables before `python start.py` |

## Tests and security scans

```bash
pip install -r requirements-dev.txt
python -m pytest tests -q                                      # 105 tests, no network / LLM needed
bandit -r agents app Talent360i_phase11C_fixed_remote_hybrid scripts start.py -x tests -ll
pip-audit -r requirements.txt                                  # see SECURITY.md for the chromadb note
```

The same checks, plus a TruffleHog secret scan, run in `.github/workflows/security.yml`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `TypeError: unsupported operand type(s) for \|` on start | Your virtualenv uses Python < 3.12. Recreate it with Python 3.12 |
| `Port 8000 is already in use` | An old `uvicorn` is still running: stop it, or set `TALENT360_API_PORT` |
| Upload says questions could not be generated | Check the LLM key in `.env`. Groq `403 Access denied` means the network blocks Groq: use `LLM_PROVIDER=cis` on the NIQ network |
| CIS calls time out | The CIS gateway is only reachable on the NIQ network / VPN |
| First `--setup` stalls at `build_chroma.py` | It is downloading the ~80 MB embedding model (one time only) |
| Forgot a password | Look in `.env`, or set a new one and run `python start.py --reseed` |

Run everything through `python start.py`. If you start uvicorn by hand, the portal must be started
**from inside its folder** (`uvicorn app:app --port 8010`). From the project root, `app` resolves to
the API package instead.

## Known limitations

See `SECURITY.md` section 5. In short, the leader *movement over time* and *item analysis* screens
still show synthetic sample figures (there is no assessment history yet); every other leader and
manager screen uses live data. The career ladder is derived from the role catalogue (next grade up in
the same tower), not from an HR succession model, and a promotion approval is only recorded here.
