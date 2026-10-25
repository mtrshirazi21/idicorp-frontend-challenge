# Front-End Engineer Assignment: IDICorp Service Console

## Handoff from the backend team

Hi — the service is ready for a browser client. We need an internal console where a signed-in user can work with sample tasks and try a few service capabilities: a calculator, a text echo, and a streamed echo. Please build the client against the API described below. The API base URL will be provided with your environment. For the local mock, the username and password default to `admin` and `admin`; an environment may provide different test credentials.

The goal is to make these flows clear and dependable for a user. We have not prescribed a framework, visual design, or application structure; choose an approach that suits the work.

## Connecting to the service

### Production addresses

- Swagger UI: [`https://ts.idicorp.ir/docs`](https://ts.idicorp.ir/docs)
- HTTP API base URL: `https://ts.idicorp.ir`
- Calculator WebSocket: `wss://ts.idicorp.ir/ws/calculator`
- Text echo WebSocket: `wss://ts.idicorp.ir/ws/echo`

Swagger's server selector uses `https://ts.idicorp.ir` for HTTP requests. The WebSocket routes are documented here with their complete secure URLs.

## API endpoints

### Sign-in endpoints

| Method | Path | What it does |
| --- | --- | --- |
| `GET` | `/api/auth/login` | Returns JSON with `captcha` (a Base64-encoded JPEG data URL in the form `data:image/jpeg;base64,<payload>`) and `salt` (the mock salt). The Base64 image payload follows the comma. Keep the returned `salt` and send that exact value in the sign-in request. |
| `POST` | `/api/auth/login` | Accepts JSON fields `username`, `password`, `salt`, and `captcha_answer`. On success, returns `access_token` and `token_type`, and sets HTTP-only `session` and `task_session` cookies. `task_session` contains a UUID bound to the signed access token and scopes task data to that login session. |

Protected HTTP endpoints need the access token as a Bearer token and the `session` cookie. Task endpoints also require the `task_session` cookie set by login. The browser must include credentials when making those requests. A failed or missing session returns `401`.

### Task endpoints

All task routes require a signed-in session.

| Method | Path | What it does |
| --- | --- | --- |
| `GET` | `/api/tasks` | Returns the signed-in session's tasks. A new task session starts with the two sample tasks. Optional `completed=true` or `completed=false` filters by completion; optional `q` searches titles and descriptions without case sensitivity. |
| `GET` | `/api/tasks/{id}` | Returns a task by ID, or `404` if that ID is not in the signed-in session's collection. |
| `POST` | `/api/tasks` | Accepts `title`, optional `description`, and optional `completed`; returns `201` with a task-shaped response. |
| `PUT` | `/api/tasks/{id}` | Replaces an existing task. Requires `title`, `description`, and `completed`; returns `404` if the task does not exist in this session. |
| `PATCH` | `/api/tasks/{id}` | Updates one or more fields of an existing task; returns `404` if it does not exist in this session. |
| `DELETE` | `/api/tasks/{id}` | Deletes an existing task and returns `204`, or `404` if it does not exist in this session. |

A task response contains `id`, `title`, `description`, `completed`, `created_at`, and `updated_at`. Titles can be up to 120 characters and descriptions up to 2,000 characters. Invalid input returns `422`; Redis unavailability returns `503`.

Tasks are stored in Redis under keys in the form `<task_session_uuid>:task_<id>`. Each task expires one hour after it is created or last updated. Set `REDIS_URL` to override the default `redis://localhost:6379/0` connection. A new successful login creates a new task session and therefore a separate task collection.

### WebSocket endpoints

The two WebSocket endpoints use different authentication flows.

#### Calculator: `/ws/calculator`

Connect to `wss://ts.idicorp.ir/ws/calculator` and send the access token as the first JSON message within one second:

```json
{"token":"<access_token>"}
```

If the token is valid, the service replies with `{"type":"ready"}`. If the token is invalid, the message is malformed, or no message arrives within one second, it sends `{"error":"Unauthorized"}` and closes the connection.

After receiving `ready`, send calculator requests with an operation and a list of numbers. The supported operations are `add`, `subtract`, `multiply`, and `divide`:

| Operation | Behavior |
| --- | --- |
| `add` | Adds all numbers in the list. |
| `subtract` | Starts with the first number and subtracts each remaining number from it. |
| `multiply` | Multiplies all numbers in the list. |
| `divide` | Starts with the first number and divides it by each remaining number in order. |

The `operation` field defaults to `add` when omitted. Provide between 1 and 100 finite numbers, each with an absolute value no greater than `1,000,000,000,000`. Division by zero is invalid.

```json
{"operation":"add","numbers":[10,20]}
```

A valid request returns `{"result": <number>}`. An invalid request returns `{"error":"Invalid calculator request"}` and leaves the connection open.

#### Text echo: `/ws/echo`

After a successful connection, it sends `{"type":"ready"}`. Then send text messages; the service echoes each message on the same connection. The connection can be used for multiple messages.

### Streaming echo endpoint: `POST /api/echo/stream`

`POST /api/echo/stream` accepts an authenticated request body and streams the same bytes back. The response uses the request's content type when provided, or `application/octet-stream` otherwise.

## What we need from the client

- A user can sign in using the CAPTCHA and credentials supplied for the environment.
- A user can browse and search tasks, filter by completion, and try creating, editing, and deleting tasks.
- A user can use each calculator operation and understand the result or an invalid request.
- A user can send multiple messages through the text echo connection and see its connection state.
- A user can send content through the streaming echo service and see the returned content.
- Loading, empty, validation, authentication, network, and service errors are communicated clearly.
- The client works on common screen sizes and can be operated with a keyboard, with clear labels and visible focus states.

The API's interactive reference is available at `/docs` on the API host. Account for the service's actual responses and the one-hour task lifetime.

## Submission

Submit the complete front-end source code to a private repository for evaluation and share that repository with `mtrshirazi21`. A built `dist`/`build` folder or other compiled assets alone is not a valid submission; evaluators must be able to inspect the source and run the application locally.

The repository must include:

- All application source files, static assets, and project configuration needed to reproduce the app.
- Dependency manifests and lock files (for example, `package.json` and the package-manager lock file).
- A README with required runtime/tool versions, install steps, any environment configuration, and the exact commands to run and build the app locally.
- An `.env.example` or equivalent configuration template with the API base URL setting, but no real secrets.

Keep generated build output optional; do not use it as a replacement for source files. The app must run as a standalone front-end without integration into the backend, and must load its assets from the front-end app or its static host rather than relying on the backend to serve or proxy them.

Document any assumptions about the service.

## Reporting a problem

If you encounter a problem or issue, open an issue in this repository and explain the problem there. We will respond to your issue in the repository.
