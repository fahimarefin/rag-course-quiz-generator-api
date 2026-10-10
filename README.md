# AI Course Generation API

An asynchronous FastAPI backend for creating AI-assisted courses. The service manages user IDs and course sessions, generates course modules with OpenAI, creates text-to-speech audio for module content, and generates quizzes with answer tracking.

## Features

- Generate, retrieve, list, and delete unique user IDs.
- Generate an AI course name and save a course configuration.
- Generate course modules one at a time from a saved course session.
- Generate and serve MP3 narration for module sections.
- Generate 10-question multiple-choice quizzes from lecture content.
- Use stored course lecture content as retrieval context for quiz generation.
- Submit answers and retrieve quiz results.
- Persist application data in MongoDB.
- Explore the API through the automatically generated Swagger UI.

## Technology

- Python 3.11+
- FastAPI and Uvicorn
- MongoDB with Motor (async MongoDB driver)
- Pydantic v2 and pydantic-settings
- OpenAI API for course content, quiz generation, and text-to-speech
- Docker and Docker Compose

## RAG-Based Educational Workflow

The project implements a retrieval-grounded generation workflow for educational content:

1. Course and module content is generated and stored in MongoDB.
2. The quiz endpoint retrieves the saved lecture document using `unique_session_id`.
3. Module introductions, topic summaries, and detailed explanations are assembled into a context prompt.
4. OpenAI generates quiz questions from that retrieved course context.

This keeps quiz generation tied to the learner's saved course instead of sending an unrelated prompt. The current implementation uses MongoDB document retrieval and prompt-based context assembly. It does not yet include embedding generation, document chunking, or a vector database. Those can be added later for semantic search across larger course or reference-document collections.

## Project Structure

```text
.
├── main.py                         # FastAPI application and lifespan setup
├── requirements.txt                # Python dependencies
├── Dockerfile
├── docker-compose.yml
├── app/
│   ├── database/database.py        # MongoDB connection and indexes
│   ├── models/                     # Database-oriented Pydantic models
│   ├── routes/                     # API route handlers
│   ├── schemas/                    # Request and response schemas
│   └── services/                   # OpenAI and domain services
└── static/audio/                   # Generated MP3 files
```

## Prerequisites

- Python 3.11 or later
- A running MongoDB instance
- An OpenAI API key for course, module, quiz, and audio generation
- Docker Desktop, if using Docker Compose

## Configuration

Create a `.env` file in the project root. The file is ignored by Git and should not be committed.

```env
MONGO_URI=mongodb://localhost:27017
DB_NAME=ScotDB
OPENAI_API_KEY=your_openai_api_key
```

`MONGO_URI` and `DB_NAME` are required when the application starts. `OPENAI_API_KEY` may be omitted for non-generation operations, but AI generation and TTS will not work without it.

## Run Locally

### Windows PowerShell

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
uvicorn main:app --reload
```

The API is available at `http://localhost:8000`.

Useful URLs:

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- Health check: `http://localhost:8000/health`

## Run with Docker Compose

```powershell
docker compose up --build
```

The API is exposed on port `8000`, and `./static` is mounted into the container so generated audio persists on the host.

Before using Compose outside local development, review `docker-compose.yml`. It currently contains a MongoDB connection in the `environment` section; replace it with a secret-managed value or remove it so the `.env` value is used. Never commit database credentials or API keys.

To stop the service:

```powershell
docker compose down
```

## API Overview

All application routes use the `/api/v1` prefix.

### Health and audio

| Method | Endpoint | Description |
| --- | --- | --- |
| GET | `/` | Returns basic service information. |
| GET | `/health` | Returns the health status. |
| GET | `/audio/{session_id}/{filename}` | Serves a generated MP3 file. |

### User IDs

| Method | Endpoint | Description |
| --- | --- | --- |
| POST | `/api/v1/user-id/generate` | Creates and stores a `USR-...` user ID. |
| GET | `/api/v1/user-id/all` | Lists all stored user IDs. |
| GET | `/api/v1/user-id/{user_id}` | Retrieves one user ID. |
| DELETE | `/api/v1/user-id/{user_id}` | Deletes one user ID. |

### Courses

| Method | Endpoint | Description |
| --- | --- | --- |
| POST | `/api/v1/course/generate` | Validates a user ID, generates a course name, and creates a `SES-...` session. |
| GET | `/api/v1/course/session/{unique_session_id}` | Retrieves one course session. |
| GET | `/api/v1/course/user/{unique_user_id}` | Retrieves all course sessions for a user. |

### Lectures and modules

| Method | Endpoint | Description |
| --- | --- | --- |
| POST | `/api/v1/course-lecture/generate-module` | Generates one module and its TTS audio. |
| GET | `/api/v1/course-lecture/{unique_session_id}` | Retrieves the saved lecture and modules. |
| DELETE | `/api/v1/course-lecture/{unique_session_id}` | Deletes a saved lecture so it can be regenerated. |
| GET | `/api/v1/course-lecture/debug/audio-files/{unique_session_id}` | Lists generated audio files for a session. |

### Quizzes

| Method | Endpoint | Description |
| --- | --- | --- |
| POST | `/api/v1/quiz/generate` | Generates or returns a quiz for a lecture session. |
| POST | `/api/v1/quiz/submit-answer` | Checks and stores an answer. Existing answers are updated. |
| GET | `/api/v1/quiz/results/{unique_session_id}/{unique_user_id}` | Returns total, answered, correct, and percentage results. |

Use `/docs` for the complete OpenAPI schema and interactive request examples.

## Typical Workflow

1. Call `POST /api/v1/user-id/generate` and save the returned `user_id`.
2. Call `POST /api/v1/course/generate` with that user ID and course configuration. Save the returned `unique_session_id`.
3. Call `POST /api/v1/course-lecture/generate-module` for each module number from `1` through `total_modules`.
4. Retrieve the finished lecture with `GET /api/v1/course-lecture/{unique_session_id}`.
5. Call `POST /api/v1/quiz/generate` with the user and session IDs.
6. Submit each answer with `POST /api/v1/quiz/submit-answer`.
7. Read the final score from `GET /api/v1/quiz/results/{unique_session_id}/{unique_user_id}`.

## Example Requests

Generate a user ID:

```powershell
curl -X POST http://localhost:8000/api/v1/user-id/generate
```

Generate one module after creating a course session:

```powershell
curl -X POST http://localhost:8000/api/v1/course-lecture/generate-module `
  -H "Content-Type: application/json" `
  -d '{"unique_session_id":"SES-ABCD12345678","module_number":1}'
```

Generate a quiz:

```powershell
curl -X POST http://localhost:8000/api/v1/quiz/generate `
  -H "Content-Type: application/json" `
  -d '{"unique_session_id":"SES-ABCD12345678","unique_user_id":"USR-..."}'
```

## Data Storage

MongoDB indexes are created during application startup for the following collections:

- `User_id table`
- `course_name_generator`
- `Course_lecture_generator`
- `Quiz_question`
- `Quiz_answer`

Generated audio is stored at:

```text
static/audio/{session_id}/module_{number}_part_{index}_{part_key}.mp3
```

Audio can be requested through `/audio/{session_id}/{filename}`. If TTS generation fails, the module is still saved with a failed voice status.

## Notes

- Quiz generation currently creates exactly 10 questions when a quiz does not already exist.
- Course modules must be generated within the configured `total_modules` range.
- The application enables permissive CORS (`*`); restrict allowed origins before production deployment.
- The Docker image starts Uvicorn with `--reload`, which is intended for development.
- No automated test suite is currently included in the repository.

## License

No license file is currently included in this repository.