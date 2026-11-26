# Receipt OCR: Backend

Kotlin / Spring Boot API that turns photographed receipts into structured, categorised expense data shared across a household. Text recognition and LLM parsing run in a separate Python service that this API orchestrates.

The React client lives in a separate repository: ([receipt-ocr-frontend](https://github.com/sziki12/receipt-ocr-frontend)).

## Features

- **Receipt scanning:** upload a photo and get back the store name, purchase date, line items and total.
- **Pluggable OCR and parsing models:** choose the OCR engine (Tesseract, PaddleOCR, or an LLM with vision) and the parsing model (GPT, Gemini, Mistral, Claude, Llama, Llava, T5) per request. This makes it easy to compare accuracy, cost and speed.
- **Total reconciliation:** if the parsed items don't add up to the receipt total, a "Parse Correction" item is added so totals stay consistent.
- **Item categorisation:** an LLM sorts items into spending categories (food, clothing, utilities and so on).
- **Households:** receipts, items and places belong to a household. Users can create households and invite others by email.
- **Statistics:** per-household spending data for charts.
- **Auth:** stateless JWT with refresh tokens, email confirmation and invitation links.
- **API docs:** Swagger UI at `/api/swagger-ui.html`.

## Architecture

```
React client ──► Spring Boot API (Kotlin) ──► Python service (OpenCV, OCR, LLM clients)
                       │                              │
                  PostgreSQL                 Tesseract / PaddleOCR / GPT / Gemini / Mistral / Claude / Llama
```

1. The client uploads an image to `POST /api/household/{householdId}/image` with the chosen OCR type, parse model and orientation.
2. The API saves a pending receipt and the image, then calls the Python service (`/ocr/process`).
3. The Python service straightens the image (edge detection, perspective transform, deskew with OpenCV), runs OCR, and extracts structured JSON with the chosen model. If the OCR type and parse model are the same LLM, one call does both.
4. The API stores items, the raw OCR output and the receipt date, then marks the receipt as no longer pending.

All data routes are household-scoped (`/api/household/{householdId}/...`). A class diagram is in [`src/main/resources/class_diagram_without_dto.png`](src/main/resources/class_diagram_without_dto.png).

### Tech stack

| Area | Technology |
| --- | --- |
| API | Kotlin 1.9, Spring Boot 3.2, Spring Data JPA / Hibernate, Java 21 |
| Security | Spring Security OAuth2 resource server, JWT (RSA-signed), java-jwt |
| Database | PostgreSQL |
| Docs | springdoc-openapi (Swagger UI) |
| Mail | Angus Mail (Gmail SMTP) |
| OCR service | Python, OpenCV, Tesseract, PaddleOCR |
| LLM clients | OpenAI, Google Gemini, Mistral, Anthropic, Ollama (Llama / Llava), Hugging Face (T5) |

## Getting started

### Prerequisites

- JDK 21
- PostgreSQL on `localhost:5432` (database/user/password `postgres`, or edit `application.properties`). `docker compose up postgres` starts one.
- Python 3.10+ with Tesseract installed, for the OCR service
- API keys for whichever LLM providers you want to use

### 1. Environment variables

The API refuses to start without these:

| Variable | Purpose |
| --- | --- |
| `SERVER_URL` | Public URL of this API (used in emailed links), e.g. `http://localhost:8080` |
| `PYTHON_URL` | URL of the Python service, e.g. `http://localhost:9090` |
| `GOOGLE_APP_EMAIL` / `GOOGLE_APP_PASSWORD` | Gmail account and app password used to send emails |

The Python service reads `GPT_API_KEY`, `GEMINI_API_KEY` and `MISTRAL_API_KEY` from `environment_variables.env`, loaded from the directory you start it in.

### 2. JWT signing keys

Keys are not committed. Generate a pair and put it in `src/main/resources/certs/`:

```bash
mkdir -p src/main/resources/certs && cd src/main/resources/certs
openssl genrsa -out keypair.pem 2048
openssl rsa -in keypair.pem -pubout -out public.pem
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in keypair.pem -out private.pem
```

### 3. Image storage

Uploaded receipts are written to `src/main/resources/image` (git-ignored, not created automatically), so create it and start the API from the repository root:

```bash
mkdir -p src/main/resources/image
```

### 4. Run

```bash
# Python OCR service (port 9090)
cd src/main/python
pip install -r Requirements.txt
python WebServerRunnable.py

# API (port 8080), from the repository root
./gradlew bootRun
```

`Requirements.txt` includes `paddlepaddle-gpu`, which needs a CUDA setup. Swap it for `paddlepaddle` on a CPU-only machine.

Docker: `./gradlew build` and then `docker compose up --build` start PostgreSQL and the API. The compose file expects `src/main/resources/env_vars.env` with the variables above, and the datasource URL must point at the `postgres` host instead of `localhost`. The Python service is not containerised.

## Status

This was my BSc thesis project. There is no automated test suite yet, and `legacy/` plus `src/main/python/Legacy/` hold an earlier subprocess-based Llama approach that the current pipeline doesn't use.
