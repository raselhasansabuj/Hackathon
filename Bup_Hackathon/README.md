# GridWise: LLM-Assisted Energy Optimization API

GridWise is a REST API that plans how a building or microgrid should use solar power, a battery, and the grid over 24 hours, so the electricity bill is as low as possible.

The special part is that the operator can write short notes in plain English, such as "Solar output will drop by half between 1 pm and 3 pm" or "Do not charge the battery between 6 pm and 9 pm". GridWise turns these notes into structured rules, checks them, and then solves the schedule with a math optimizer.
GridWise is a FastAPI service that generates optimal 24-hour energy dispatch schedules for a solar + battery + grid setup. It combines natural-language operator directives with a linear programming solver to produce a cost-minimizing hourly plan.

**Version: 3.1.0**

🟢 **Live and deployed on Render:**

| Resource | URL |
| :--- | :--- |
| Base URL | [https://bup-cse-fest-2026-grid.onrender.com](https://bup-cse-fest-2026-grid.onrender.com) |
| Health check | [https://bup-cse-fest-2026-grid.onrender.com/health](https://bup-cse-fest-2026-grid.onrender.com/health) |
| Interactive API docs (Swagger UI) | [https://bup-cse-fest-2026-grid.onrender.com/docs](https://bup-cse-fest-2026-grid.onrender.com/docs) |

> **Note:** this runs on Render's free tier, so the service spins down after periods of inactivity. The first request after idle time may take 30–60 seconds to respond while it wakes up.

## How It Works

The request goes through three steps:

1. **Interpret the notes.** Each operator note is converted into a structured directive. If an LLM API key is set, the LLM path is tried first. Otherwise a rule-based interpreter reads the notes.
2. **Validate the directives.** A guardrail layer cleans every directive: hours are limited to 0-23, factors to 0-1, and battery reserves to the battery capacity. Anything unknown becomes a no-op.
3. **Optimize.** A linear program (PuLP with the CBC solver) chooses grid import, solar use, and battery charge or discharge for each hour to minimize total cost in BDT.

## Tech Stack

- Python 3.10
- FastAPI and Uvicorn
- Pydantic v2 for request and response validation
- PuLP (CBC solver) for optimization
- Docker and Vercel for deployment

## Project Structure

```
.
├── main.py             # API, directive interpreter, validator, optimizer
├── requirements.txt    # Python dependencies
├── Dockerfile          # Container build
├── vercel.json         # Vercel deployment config
└── sample_request.json # Example request for testing
```

## Run Locally

```bash
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

The interactive API docs are available at `http://localhost:8000/docs`.

## Run With Docker

```bash
docker build -t gridwise .
docker run -p 8000:8000 gridwise
```

## Deploy on Vercel

The included `vercel.json` routes every request to `main.py` using the `@vercel/python` builder. Import the repository in Vercel and deploy.

## API Endpoints

### GET /health

Returns `{"status": "ok"}`.

### POST /optimize-energy

Creates the optimized 24-hour plan.

Try it with the sample file:

```bash
curl -X POST http://localhost:8000/optimize-energy \
  -H "Content-Type: application/json" \
  -d @sample_request.json
```

**Request body**

| Field | Type | Description |
|-------|------|-------------|
| `scenario_id` | string | Name or ID of the scenario |
| `operator_notes` | list of strings | 1 to 3 plain-English notes |
| `hours` | list of 24 objects | One entry per hour: `hour`, `demand_kwh`, `solar_kwh`, `tariff_bdt_per_kwh` |
| `battery` | object | `capacity_kwh`, `initial_energy_kwh`, `minimum_energy_kwh`, `max_charge_kwh_per_hour`, `max_discharge_kwh_per_hour` |

**Response body**

| Field | Description |
|-------|-------------|
| `scenario_id` | Same ID as the request |
| `directive_interpretation` | How each note was understood |
| `hourly_plan` | For each hour: `grid_kwh`, `solar_used_kwh`, `battery_action` (charge, discharge, idle), `battery_kwh`, `battery_energy_after_kwh` |
| `total_grid_kwh` | Total energy imported from the grid |
| `total_cost_bdt` | Total cost in BDT |
| `peak_grid_kwh` | Highest hourly grid import |
| `plan_summary` | Short text summary |

**Error codes**

| Code | Meaning |
|------|---------|
| 400 | Request is malformed or has the wrong structure |
| 422 | The schedule is infeasible with the given rules |

## Supported Operator Directives

| Directive | Example note | Effect |
|-----------|--------------|--------|
| `solar_reduction` | "Solar output will drop by half between 1 pm and 3 pm" | Multiplies solar in those hours by a factor |
| `minimum_battery_reserve` | "Keep at least 90 kWh in the battery from 6 pm to 9 pm" | Raises the minimum battery level in those hours |
| `no_charge_window` | "Do not charge the battery between 6 pm and 9 pm" | Battery charging is set to zero in those hours |
| `no_discharge_window` | "The battery must not discharge between 2 pm and 4 pm" | Battery discharging is set to zero in those hours |
| `max_grid_window` | "Grid import must stay at or below 155 kWh from 6 pm to 8 pm" | Caps grid import in those hours |
| `no_op` | Any note that does not affect the schedule | Ignored |

## Optimization Model

- **Goal:** minimize the sum of `grid_kwh x tariff` over 24 hours
- **Energy balance each hour:** grid + solar used + battery discharge = demand + battery charge
- **Solar:** solar used cannot exceed available solar
- **Battery:** stays between the minimum reserve and the capacity, and respects the hourly charge and discharge limits
- **Daily cycle:** the battery ends hour 23 with the same energy it started with

## LLM Configuration (Optional)

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Enables the LLM call to the Anthropic API |
| `OPENAI_API_KEY` | Reserved for an OpenAI integration |
| `LLM_MODEL` | Model name to use (default: `claude-sonnet-4-6`) |

The LLM step is a starting hook. The LLM reply is not parsed into directives yet, so the rule-based interpreter currently produces the final directives, and the guardrail layer always runs before the optimizer.

## Known Limitations

- The rule-based interpreter understands a fixed set of time phrases (for example "1 pm to 3 pm"). If no time window is found, it defaults to hours 13 and 14.
- Notes are matched by keywords, so very unusual wording may be treated as a no-op.

## Author

Built by [raselhasansabuj](https://github.com/raselhasansabuj).
