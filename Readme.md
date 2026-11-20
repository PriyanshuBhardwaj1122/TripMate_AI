# TripMate AI

A multi-agent travel planner. You describe a trip in plain language, and five agents working through a LangGraph workflow return a structured plan with flight guidance, hotel options, weather, a day-by-day itinerary and a budget estimate.

Planning a trip usually means jumping between separate sites for flights, hotels and weather, then stitching everything together by hand. TripMate does that in one request.

## How it works

The backend is a LangGraph `StateGraph` with five nodes that run in sequence. Each node reads from and writes to a shared state, and the full state is saved to PostgreSQL after every step.

```mermaid
flowchart LR
    A[User request] --> B[Flight agent]
    B --> C[Hotel agent]
    C --> D[Weather agent]
    D --> E[Itinerary agent]
    E --> F[Response agent]
    F --> G[Final plan]
```

| Agent | What it does | Data source |
|---|---|---|
| Flight | Looks up airport and airline data, then asks the LLM for likely routes, airlines, duration, fare range and booking advice | AviationStack via MCP |
| Hotel | Searches for hotels matching the request | Tavily via MCP |
| Weather | Extracts the destination city, then fetches current weather and a forecast | Custom weather MCP server |
| Itinerary | Combines flights, hotels and weather into a practical, budget-aware itinerary | LLM |
| Response | Formats the final answer: trip summary, flights, hotels, weather, day-by-day plan, estimated budget, recommendations | LLM |

**Note on flight data:** the flight agent works from airport and airline listings, so it gives route guidance and typical fare ranges rather than live ticket prices.

## Tech stack

- **Orchestration:** LangGraph, LangChain
- **LLM:** Groq (`openai/gpt-oss-20b`) through `langchain-groq`
- **Tool integrations:** Model Context Protocol (MCP): AviationStack, remote Tavily search, and a custom weather server
- **State persistence:** PostgreSQL through LangGraph's `PostgresSaver` checkpointer
- **API:** FastAPI
- **Language:** Python

## State and persistence

Every run gets a `thread_id`. If you don't pass one, the backend generates it. LangGraph checkpoints the state (messages, flight, hotel and weather results, itinerary, LLM call count) to PostgreSQL after each node, so a conversation can be traced and resumed by its `thread_id`.

The entry point used by the API layer is:

```python
run_travel_agent(user_input: str, thread_id: str | None = None)
```

It returns:

```json
{
  "thread_id": "user_...",
  "answer": "final formatted plan",
  "flight_results": "...",
  "hotel_results": "...",
  "weather_results": "...",
  "itinerary": "...",
  "llm_calls": 3
}
```

## Getting started

### Prerequisites

- Python 3.10 or newer
- A PostgreSQL database (a hosted one such as Render works; the connection uses `sslmode=require`)
- API keys for Groq, Tavily and AviationStack

### Setup

```bash
git clone https://github.com/PriyanshuBhardwaj1122/tripmate-ai.git
cd tripmate-ai

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

Copy the example environment file and fill in your own values:

```bash
cp .env.example .env
```

```env
GROQ_API_KEY=
DATABASE_URL=
TAVILY_API_KEY=
AVIATIONSTACK_API_KEY=
```

The app will not start if `GROQ_API_KEY` or `DATABASE_URL` is missing. Never commit `.env`.

### Run

```bash
uvicorn <your_main_file>:app --reload
```

Then open `http://127.0.0.1:8000/docs` for the interactive API docs.

## Example request

> "Plan a 4-day trip from Delhi to Goa in December on a mid-range budget."

The response includes a trip summary, flight guidance, hotel suggestions, a weather outlook with packing advice, a day-by-day itinerary, an estimated budget and final recommendations.

## Project structure

```
tripmate-ai/
├── <graph file>        # LangGraph workflow: state, five agents, PostgreSQL checkpointer
├── mcp_client.py       # MCP clients for Tavily, AviationStack and the weather server
├── <api file>          # FastAPI app that calls run_travel_agent
├── requirements.txt
├── .env.example
└── README.md
```

## Known limitations

- Agents run one after another, so a request takes as long as all five steps combined.
- Flight results are guidance based on airport and airline data, not live prices.
- The weather agent finds the destination city from the request text, so a very vague request can return poor weather data.

## Possible improvements

- Run the flight, hotel and weather agents in parallel to cut response time
- Add tests and a CI pipeline with GitHub Actions
- Add a simple frontend and a live demo

## Author

Priyanshu Bhardwaj: [GitHub](https://github.com/PriyanshuBhardwaj1122) | [LinkedIn](https://www.linkedin.com/in/priyanshu-bhardwaj-890b23371/)
