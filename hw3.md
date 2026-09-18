# Homework 3: Test, Containerize, and Deploy an AI-Assisted App

In this homework, you'll deploy Agent Relay. It's is a small messaging system for software agents.

An agent sends a task to another agent, a worker claims the task, and the worker acknowledges the result. The database stores the messages and their delivery attempts. A small dashboard lets you watch the message lifecycle.

You'll use your coding agent to test and containerize Agent Relay. Then you'll deploy it to a local Kubernetes cluster with kind (Kubernetes in Docker).

The starter project already contains the API and dashboard.

You don't need a cloud account, LLM API key, or external message broker. Everything runs on your machine. Ask your agent to install and verify each tool when you need it.

## Environment preparation

Ensure the VM has Git, Python, and Docker installed. Run:

```
cat /etc/os-release
git --version
python3 --version
docker --version
docker compose version
```

## Clone repository

```
mkdir -p ~/projects
cd ~/projects

git clone https://github.com/ketut-garjita/agent-relay.git
cd agent-relay

git remote -v
ls -la
```

Check the branch and project structure:

```
git branch --show-current
find . -maxdepth 2 -type f | sort
```

## Read the project documentation

Before running the application, read the README and the acceptance test specifications:

```
cat README.md
cat SPEC.md
```

## Question 1: Understand the project

Fork the Agent Relay starter repository from here.

Ask your agent to run the project. Try to understand it and experiment with it.

Which description matches the project's architecture? (1 point)

- Agents exchange tasks directly with each other.
- Agents claim tasks from a DB through an HTTP API.
- Agents consume tasks from a message broker.
- The browser stores and executes tasks.

  Solution:
  ==> Understanding the Agent Relay architecture

  The application's main components.

  <img width="656" height="379" alt="image" src="https://github.com/user-attachments/assets/ce9005d5-d3b0-4fa5-8998-b1821affefa0" />

  The application uses SQLite for local storage and FastAPI as the communication intermediary.
  
  Workers do not retrieve tasks directly from the sending agent, and the starter project does not use an external message broker.

  Run the application locally
  
  Install dependencies  
  ```
  uv sync
  ```

  Run the API
  ```
  uv run uvicorn main:app --reload
  ```

  Keep this terminal running.

  Open a second terminal in the VM and verify:
  ```
  curl -i http://127.0.0.1:8000/health
  curl -i http://127.0.0.1:8000/ready
  ```
  ```
  $ curl -i http://127.0.0.1:8000/health
  HTTP/1.1 200 OK
  date: Thu, 17 Sep 2026 04:50:13 GMT
  server: uvicorn
  content-length: 15
  content-type: application/json
  ```
  ```
  $ curl -i http://127.0.0.1:8000/ready
  HTTP/1.1 200 OK
  date: Thu, 17 Sep 2026 04:50:35 GMT
  server: uvicorn
  content-length: 18
  content-type: application/json
  ```

  Then, open the dashboard via the VM's browser: [http://127.0.0.1:8000/](http://127.0.0.1:8000/)

  ### Answer to Question 1: Agents claim tasks from a database via an HTTP API.


---
## Question 2: Register agents and test the task flow

Ask your coding agent to read SPEC.md (in the starter repo root) and try its first acceptance scenario with your local Agent Relay:

Register two agents and have them exchange a task and its result.

Check the result in the dashboard. Then ask your coding agent to turn this flow into an API integration test against the real API and DB. Run the test and confirm it passes.

Which task status does the sender see after the recipient submits its result? (1 point)
- queued
- processing
- completed
- delivered

Solution:

**Flow**

```
Alice
  │
  │ POST /tasks
  ▼
Agent Relay + SQLite
  │
  │ claim
  ▼
Bob / Worker
  │
  │ complete
  ▼
Agent Relay
  │
  ▼
Alice → status: completed
```

1. 1. Use the obtained token

Set the environment variable in the same terminal. Since the previous token has already been exposed, for security reasons, it is better to re-register the two agents and use a new token.

  ```
  export ALICE_TOKEN='<ALICE token>'
  export BOB_TOKEN='<BOB token>'
  ```

  ```
   alice=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
    -H 'content-type: application/json' \
    -d '{"name":"alice-test"}')
  
  bob=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/agents \
    -H 'content-type: application/json' \
    -d '{"name":"uppercase-test"}')
  
  export ALICE_ID=$(echo "$alice" | python3 -c 'import sys,json; print(json.load(sys.stdin)["agent_id"])')
  export ALICE_TOKEN=$(echo "$alice" | python3 -c 'import sys,json; print(json.load(sys.stdin)["token"])')
  
  export BOB_ID=$(echo "$bob" | python3 -c 'import sys,json; print(json.load(sys.stdin)["agent_id"])')
  export BOB_TOKEN=$(echo "$bob" | python3 -c 'import sys,json; print(json.load(sys.stdin)["token"])')
  
  echo "Alice ID: $ALICE_ID"
  echo "Bob ID:   $BOB_ID"
  ```

2. Alice sends a task to Bob

We use a simple input:
```
hello agent relay
```

Run:
```
task=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/tasks \
  -H "Authorization: Bearer $ALICE_TOKEN" \
  -H 'content-type: application/json' \
  -d "{\"to\":\"$BOB_ID\",\"input\":\"hello agent relay\"}")

echo "$task"
```

Save:
```
export TASK_ID=$(echo "$task" | python3 -c 'import sys,json; print(json.load(sys.stdin)["task_id"])')

echo "Task ID: $TASK_ID"
```

3. Bob claims the task

Now Bob retrieves the task from the database via the API:

```
claim=$(curl -sS -X POST http://127.0.0.1:8000/api/v1/tasks/claim \
  -H "Authorization: Bearer $BOB_TOKEN" \
  -H 'content-type: application/json' \
  -d '{"worker_id":"vm-worker-1","wait_seconds":0}')

echo "$claim"
```
{"task_id":"task_24210f12cb244e4a89b14f6f2bdc64fc","from":"agent_4df6038cb4964d76839b95c6f4a9f5bb","input":"hello agent relay","attempt":2,"claim_token":"clm_q_siPB-qGpwxJCpXqoikN6qoU0yEbyHsMn-wV1UDjow","lease_expires_at":"2026-09-17T07:29:45Z"}

The claim response will provide a claim token.

Save the token:
```
export CLAIM_TOKEN=$(echo "$claim" | python3 -c 'import sys,json; print(json.load(sys.stdin)["claim_token"])')
echo "Claim received."
```

Perhatikan bahwa pada titik ini task seharusnya berada pada status:
```
processing
```

4. Bob sends the results

```
curl -sS -X POST \
  "http://127.0.0.1:8000/api/v1/tasks/$TASK_ID/complete" \
  -H "Authorization: Bearer $BOB_TOKEN" \
  -H 'content-type: application/json' \
  -d "{\"claim_token\":\"$CLAIM_TOKEN\",\"output\":\"HELLO AGENT RELAY\"}"
```

5. Alice memeriksa task

Now use Alice's token:
```
curl -sS \aeng@linuxmint-vm:~/projects/zoomcamp/myprojects/agent-relay$ curl -sS \
  "http://127.0.0.1:8000/api/v1/tasks/$TASK_ID" \
  -H "Authorization: Bearer $ALICE_TOKEN"
```

{"task_id":"task_24210f12cb244e4a89b14f6f2bdc64fc","from":"agent_4df6038cb4964d76839b95c6f4a9f5bb","to":"agent_351dc611a800453cab6be16e6ae9d1c1","input":"hello agent relay","status":"completed","output":"HELLO AGENT RELAY","error":null,"attempt_count":2,"created_at":"2026-09-17T05:49:56Z","finished_at":"2026-09-17T07:29:00Z"}(base) 

The task should demonstrate:
```
status = completed
```

The lifecycle flow is:
```
queued
↓
processing
↓
completed
```
Thus, the sender sees the "completed" status after the recipient/worker successfully submits the result.

5. Add a test

Add the following function at the end of the test_agent_relay.py file

```
def test_acceptance_task_flow_sender_sees_completed():
    with TestClient(main.app) as client:
        sender, sender_headers = register(client, "alice")
        recipient, recipient_headers = register(client, "uppercase")

        sent = client.post(
            "/api/v1/tasks",
            headers=sender_headers,
            json={
                "to": recipient["agent_id"],
                "input": "hello agent relay",
            },
        )

        assert sent.status_code == 201
        task_id = sent.json()["task_id"]

        claim = client.post(
            "/api/v1/tasks/claim",
            headers=recipient_headers,
            json={
                "worker_id": "integration-test-worker",
                "wait_seconds": 0,
            },
        )

        assert claim.status_code == 200
        claim_data = claim.json()
        assert claim_data["task_id"] == task_id

        complete = client.post(
            f"/api/v1/tasks/{task_id}/complete",
            headers=recipient_headers,
            json={
                "claim_token": claim_data["claim_token"],
                "output": "HELLO AGENT RELAY",
            },
        )

        assert complete.status_code == 200

        result = client.get(
            f"/api/v1/tasks/{task_id}",
            headers=sender_headers,
        )

        assert result.status_code == 200
        task_data = result.json()

        assert task_data["status"] == "completed"
        assert task_data["output"] == "HELLO AGENT RELAY"
        assert task_data["attempt_count"] == 1
```
6. Run the test
  ```
  uv run pytest -q
  ```
  ```                                                                                                                                                           [100%]
  ========================================================================== warnings summary ===========================================================================
  .venv/lib/python3.11/site-packages/fastapi/testclient.py:1
    /home/dataeng/projects/zoomcamp/myprojects/agent-relay/.venv/lib/python3.11/site-packages/fastapi/testclient.py:1: StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.
      from starlette.testclient import TestClient as TestClient  # noqa
  
  .venv/lib/python3.11/site-packages/starlette/testclient.py:53
    /home/dataeng/projects/zoomcamp/myprojects/agent-relay/.venv/lib/python3.11/site-packages/starlette/testclient.py:53: DeprecationWarning: The anyio.abc.BlockingPortal alias is deprecated, use anyio.from_thread.BlockingPortal instead.
      _PortalFactoryType = Callable[[], AbstractContextManager[anyio.abc.BlockingPortal]]
  
  -- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
  5 passed, 2 warnings in 4.38s
  ```

Question 2 is complete:
```text
✅ Two agents successfully registered.
✅ Sender created a task via HTTP API.
✅ Recipient claimed the task via HTTP API.
✅ Recipient submitted the result via HTTP API.
✅ Sender can see the task as completed.
✅ "HELLO AGENT RELAY" output is stored.
✅ Acceptance flow automated as an integration test.
✅ Test uses the API and an actual SQLite database for the test environment.
```
### Answer to Question 2: completed

          
## Question 3: Containerization

Ask your coding agent to create a Dockerfile for Agent Relay. Build the image as agent-relay:local and run it with the API port published to your machine.

Tip: run uvicorn with --host 0.0.0.0 inside the container, otherwise -p looks broken (uvicorn defaults to 127.0.0.1).

Open the dashboard and repeat the task flow from Question 2 against the containerized API.

Which Docker option publishes a container's port to your machine?
  - --expose
  - -p
  - -v
  - --name

Solution:

==> Dockerize Agent Relay

Now, let's create a Dockerfile, build the image, and then run the container.

Before modifying the file, first check if the repository already has a Dockerfile:

```
ls -la Dockerfile docker-compose.yml 2>/dev/null
```
Make Dockerfile:
```
FROM python:3.11-slim

WORKDIR /app

RUN pip install --no-cache-dir uv

COPY pyproject.toml uv.lock ./

RUN uv sync --frozen

COPY . .

EXPOSE 8000

CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Then build:
```
docker build -t agent-relay:local .
```

Once finished, check:
```
docker images | grep agent-relay
```

Then run:
```
docker run --rm \
  --name agent-relay \
  -p 8000:8000 \
  agent-relay:local
```

At the other terminal, verification:
```
curl http://localhost:8000/health
```

Estimated at approximately:
```
{"status":"ok"}
```

Then:
```
curl http://localhost:8000/ready
```

If both succeed, we have proven:

source code → Docker image → container → published port → HTTP API

| Validation | Results |
|------------|---------|
| docker build | ✅ Success |
| Image agent-relay:local | ✅ 361 MB |
| Container port 8000 published | ✅ |
| GET /health | ✅ {"status":"ok"} |
| GET /ready | ✅ {"status":"ready"} |


So the flow is proven:
```text
Python source
    ↓
Dockerfile
    ↓
agent-relay:local
    ↓
Docker container
    ↓
localhost:8000
    ↓
/health + /ready
```
### Answer to Question 3: -p


## Question 4: Docker Compose and PostgreSQL

Ask your coding agent to replace SQLite with PostgreSQL and create a compose.yaml that runs Agent Relay and PostgreSQL together. Name the database service postgres.

Start the stack:
```
docker compose up --build
```

Run the integration test from Question 2 against the Compose stack and check the result in the dashboard. Confirm that the app stores its data in PostgreSQL.

Which hostname should the API use to connect to the postgres service in Docker Compose?
- localhost
- postgres
- host.docker.internal
- 0.0.0.0

Solution:

Q4 — PostgreSQL + Docker Compose

Now we move on to a more important step: replacing the SQLite database with PostgreSQL and running Agent Relay alongside PostgreSQL using Docker Compose.

The target architecture is as follows:

┌──────────────────────┐
│     agent-relay       │
│      FastAPI          │
│       :8000           │
└──────────┬───────────┘
│
PostgreSQL
postgres:5432
│
┌──────────▼───────────┐
│     persistent        │
│       volume          │
└──────────────────────┘

The key requirement for Q4 is that Agent Relay must no longer rely on SQLite when running via Compose.

Before creating the `docker-compose.yml` file, first check the database configuration in the source code:
```
grep -R "RELAY_DATABASE_URL\|DATABASE_URL\|sqlite\|postgres" -n \
  main.py database.py storage.py pyproject.toml
```

```text
database.py:22:    return os.getenv("RELAY_DATABASE_URL") or os.getenv("DATABASE_URL") or "sqlite:///./agent-relay.db"
database.py:33:DATABASE_URL = _database_url()
database.py:133:def _is_sqlite(url: str) -> bool:
database.py:134:    return url.startswith("sqlite")
database.py:138:if _is_sqlite(DATABASE_URL):
database.py:140:    if DATABASE_URL in {"sqlite://", "sqlite:///:memory:"}:
database.py:145:engine: Engine = create_engine(DATABASE_URL, **engine_kwargs)
database.py:147:if _is_sqlite(DATABASE_URL):
database.py:150:    def _sqlite_pragmas(dbapi_connection: Any, _connection_record: Any) -> None:
database.py:246:    "DATABASE_URL",
```

So, Docker Compose can use RELAY_DATABASE_URL to point the Agent Relay to PostgreSQL.

Before creating the Compose configuration, check the database dependencies in pyproject.toml:
```
grep -n -A20 -B5 "dependencies" pyproject.toml
```

We need to ensure that a PostgreSQL driver—such as `psycopg` or `psycopg2`—is available. If it is not already present, add the appropriate dependency.

Also, run:
```
sed -n '1,180p' database.py
```

Focus on the section:
```
_database_url()
create_engine(...)
```

Target connection string Compose

Later the environment variables in the Agent Relay service will be in the form:

```yaml
environment: 
RELAY_DATABASE_URL: postgresql+psycopg://agent_relay:agent_relay_password@postgres:5432/agent_relay
```

Pay attention to the hostname:
```
postgres
```

That is the PostgreSQL service name in Docker Compose, not localhost.

Once the dependencies are verified, we will create:
- postgres service
- agent-relay service
- named volume for PostgreSQL
- PostgreSQL healthcheck
- agent-relay dependency on a healthy PostgreSQL instance
- Agent Relay port mapping (8000:8000)

The PostgreSQL dependency is already available:
```
"psycopg[binary]>=3.3.5"
```

And database.py already supports PostgreSQL URLs via:
```python
RELAY_DATABASE_URL
```

However, there is an important note: comments in the source code indicate that the PostgreSQL implementation is a planned exercise. Additionally, the `immediate_transaction()` function likely still relies on a mechanism specific to SQLite. Before using Compose, check that section:

```
sed -n '180,280p' database.py
```

We need to ensure that the task claim code does not use SQLite SQL, such as:
```SQL
BEGIN IMMEDIATE
```

PostgreSQL does not support that syntax. If it is still used, Compose might start successfully, but the endpoint claim will fail.

Run:
```
sed -n '180,280p' database.py
```

After that, we will determine whether simply creating a `docker-compose.yml` file is sufficient, or if minor changes are needed so that PostgreSQL transactions use row-level locking (`SELECT ... FOR UPDATE`) alongside standard transactions.

Currently, immediate_transaction() always executes:

connection.exec_driver_sql("BEGIN IMMEDIATE")

That is only valid for SQLite. We need to branch the transaction logic:

- SQLite → continue using BEGIN IMMEDIATE
- PostgreSQL → use a standard SQLAlchemy transaction

Change the immediate_transaction() function to:
```
@contextmanager
def immediate_transaction() -> Generator[Session, None, None]:
    """Run an atomic transaction for claims, recovery, and terminal actions.

    SQLite uses BEGIN IMMEDIATE to serialize writers.
    PostgreSQL uses the normal SQLAlchemy transaction and relies on
    row-level locking in the storage layer.
    """

    if _is_sqlite(DATABASE_URL):
        connection = engine.connect()
        session = Session(bind=connection, expire_on_commit=False, autoflush=True)
        try:
            connection.exec_driver_sql("BEGIN IMMEDIATE")
            yield session
            session.flush()
            connection.commit()
        except Exception:
            connection.rollback()
            raise
        finally:
            session.close()
            connection.close()
    else:
        with SessionLocal.begin() as session:
            yield session
```

However, this change alone is not sufficient for PostgreSQL concurrency if the claim query in `storage.py` still does not use row locks. Check the claim section:

```bash
grep -n -A100 -B20 "claim" storage.py
```

Find the query that selects tasks with a 'queued' status. For PostgreSQL, the query needs to use:
```bash
.with_for_update(skip_locked=True)
```

Example pattern:
```
statement = (
    select(Task)
    .where(
        Task.recipient_id == recipient_id,
        Task.status == "queued",
    )
    .order_by(Task.created_at, Task.id)
    .with_for_update(skip_locked=True)
    .limit(1)
)
```

For SQLite compatibility, `with_for_update(skip_locked=True)` can be called because SQLite will ignore that clause when generating the SQL.

From storage.py, the claim query currently does not use row-level locking:
```python
select(Task)
.where(Task.recipient_id == agent_id, Task.status == "queued")
.order_by(Task.created_at, Task.id)
.limit(1)
```

For PostgreSQL, change that section to:
```
task = db.scalar(
    select(Task)
    .where(
        Task.recipient_id == agent_id,
        Task.status == "queued",
    )
    .order_by(Task.created_at, Task.id)
    .with_for_update(skip_locked=True)
    .limit(1)
)
```

So, the `claim_one()` function in the query section becomes:
```
def claim_one(agent_id: str, worker_id: str | None) -> dict[str, Any] | None:
    with immediate_transaction() as db:
        now = utcnow()
        recover_expired_in_session(db, now)

        task = db.scalar(
            select(Task)
            .where(
                Task.recipient_id == agent_id,
                Task.status == "queued",
            )
            .order_by(Task.created_at, Task.id)
            .with_for_update(skip_locked=True)
            .limit(1)
        )

        if task is None:
            return None

        if task.attempt_count >= MAX_ATTEMPTS:
            task.status = "failed"
            task.error = "attempts_exhausted"
            task.finished_at = as_db_time(now)
            return None

        claim_token = new_secret("clm")
        task.status = "processing"
        task.attempt_count += 1
        lease_expires = as_db_time(now + timedelta(seconds=LEASE_SECONDS))

        db.add(
            Attempt(
                task_id=task.id,
                attempt_number=task.attempt_count,
                worker_id=worker_id,
                claim_token_hash=secret_hash(claim_token),
                claimed_at=as_db_time(now),
                lease_expires_at=lease_expires,
                finished_at=None,
                outcome="processing",
                terminal_action=None,
                terminal_payload_hash=None,
            )
        )

        db.flush()

        return {
            "task_id": task.id,
            "from": task.sender_id,
            "input": task.input,
            "attempt": task.attempt_count,
            "claim_token": claim_token,
            "lease_expires_at": iso_time(lease_expires),
        }
```

Then change immediate_transaction() in database.py to not run BEGIN IMMEDIATE on PostgreSQL:
```python
@contextmanager
def immediate_transaction() -> Generator[Session, None, None]:
    """Run an atomic transaction for task operations."""

    if _is_sqlite(DATABASE_URL):
        connection = engine.connect()
        session = Session(bind=connection, expire_on_commit=False, autoflush=True)

        try:
            connection.exec_driver_sql("BEGIN IMMEDIATE")
            yield session
            session.flush()
            connection.commit()
        except Exception:
            connection.rollback()
            raise
        finally:
            session.close()
            connection.close()
    else:
        with SessionLocal.begin() as session:
            yield session
```

After making those two changes, run the local tests:
```
uv run pytest -q
```
5 passed, 2 warnings in 4.13s

PostgreSQL-compatible changes no longer break SQLite tests:
- BEGIN IMMEDIATE is still used for SQLite.
- PostgreSQL will use regular SQLAlchemy transactions.
- The claim query already uses with_for_update(skip_locked=True).
- All five tests still passed.

Next: create docker-compose.yml

Create the following files in the project root:
```
services:
  postgres:
    image: postgres:16-alpine
    container_name: agent-relay-postgres
    environment:
      POSTGRES_DB: agent_relay
      POSTGRES_USER: agent_relay
      POSTGRES_PASSWORD: agent_relay_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U agent_relay -d agent_relay"]
      interval: 5s
      timeout: 5s
      retries: 10

  agent-relay:
    build:
      context: .
      dockerfile: Dockerfile
    image: agent-relay:compose
    container_name: agent-relay-api
    environment:
      RELAY_DATABASE_URL: postgresql+psycopg://agent_relay:agent_relay_password@postgres:5432/agent_relay
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8000:8000"
volumes:
  postgres_data:
```

Run:
```
docker compose up --build
```

In another terminal, check:
```
curl http://localhost:8000/health

curl http://localhost:8000/ready
```

Expected:
```
{"status":"ok"}

{"status":"ready"}
```

Check running container and stop
```
dockse ps
docker stop agent-relay
```

Restart docker compose:
```
docker compose down
docker compose up -d
```

Then, verify the database used by the container:
```
docker compose exec agent-relay sh -lc 'echo "$RELAY_DATABASE_URL"'
```
postgresql+psycopg://agent_relay:agent_relay_password@postgres:5432/agent_relay

Database Hostname : postgres

This proves that the Agent Relay is running in Compose and connecting to PostgreSQL via the Compose service name.

### Answer to Question 4: postgres

## Question 5: Deploy to Kubernetes

Ask your coding agent to install kind and kubectl if needed, then create a local Kubernetes cluster.

Create manifests in k8s/ for Agent Relay and PostgreSQL, including Services, persistent DB storage, and readiness checks. Load your Docker image into kind and deploy the application.

Check that the pods are ready. Open the dashboard through port forwarding and verify the task flow from Question 2.

Which Kubernetes resource keeps the requested number of application replicas running and manages updates?
`- Service`
`- ConfigMap`
`- Deployment`
`- Secret`

Solution:


