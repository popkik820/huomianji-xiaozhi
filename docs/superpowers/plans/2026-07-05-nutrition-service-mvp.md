# Nutrition Service MVP Implementation Plan

> For agentic workers: implement task-by-task. This plan intentionally omits Git steps per user instruction.

**Goal:** Build a local `nutrition-service` MVP with FastAPI endpoints and a CLI for entering food intake text.

**Architecture:** Create an independent Python package under `nutrition-service/`. Core parsing, recommendation, and storage logic stay framework-independent; FastAPI and CLI reuse the same service layer.

**Tech Stack:** Python 3.10, FastAPI, Uvicorn, SQLite, standard-library `unittest`.

---

## File Structure

- `nutrition-service/nutrition_service/models.py`: dataclasses and constants.
- `nutrition-service/nutrition_service/parser.py`: rule-based MVP parser with a future LLM parser boundary.
- `nutrition-service/nutrition_service/recommender.py`: deterministic recommendation rules.
- `nutrition-service/nutrition_service/storage.py`: SQLite daily food record storage.
- `nutrition-service/nutrition_service/service.py`: orchestration used by API and CLI.
- `nutrition-service/nutrition_service/api.py`: FastAPI app.
- `nutrition-service/nutrition_cli.py`: local terminal input.
- `nutrition-service/tests/`: unit and API tests.

## Tasks

### Task 1: Core Parser And Recommendation

- Write tests for parsing common Chinese foods and extracting dough weight.
- Implement the rule parser.
- Write tests for 24% water ratio and coarse-grain recommendation.
- Implement the recommender.

### Task 2: Storage And Service Layer

- Write tests for saving and querying same-day food records.
- Implement SQLite storage.
- Write tests for parse-confirm-recommend orchestration.
- Implement service layer.

### Task 3: FastAPI And CLI

- Write API tests for `/api/v1/intake/parse`, `/api/v1/recommendations/coarse-grain`, and `/api/v1/intake/today`.
- Implement FastAPI app.
- Write CLI smoke test against service layer.
- Implement CLI.

### Task 4: Verification

- Run `D:\conda\envs\xiaozhi-esp32-server\python.exe -m unittest discover -s nutrition-service/tests -v`.
- Run a local API smoke test if Uvicorn can start in the current environment.
- Run a CLI sample input and inspect JSON output.
