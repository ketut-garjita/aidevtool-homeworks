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

---
## Question 1: Understand the project

Fork the Agent Relay starter repository from here.

Ask your agent to run the project. Try to understand it and experiment with it.

Which description matches the project's architecture? (1 point)

- Agents exchange tasks directly with each other.
- Agents claim tasks from a DB through an HTTP API. ✅
- Agents consume tasks from a message broker.
- The browser stores and executes tasks.

  #### SOLUTION
  
  ==> Understanding the Agent Relay architecture

  The application's main components.

  ![agent-relay-local-architecture](assets/agent-relay-local-architecture.png)
  
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
- completed ✅
- delivered

#### SOLUTION

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

1. Use the obtained token

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
Note that at this point, the task should be in the following status:
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

---          
## Question 3: Containerization

Ask your coding agent to create a Dockerfile for Agent Relay. Build the image as agent-relay:local and run it with the API port published to your machine.

Tip: run uvicorn with --host 0.0.0.0 inside the container, otherwise -p looks broken (uvicorn defaults to 127.0.0.1).

Open the dashboard and repeat the task flow from Question 2 against the containerized API.

Which Docker option publishes a container's port to your machine?
  - --expose
  - -p ✅
  - -v
  - --name

#### SOLUTION

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

---
## Question 4: Docker Compose and PostgreSQL

Ask your coding agent to replace SQLite with PostgreSQL and create a compose.yaml that runs Agent Relay and PostgreSQL together. Name the database service postgres.

Start the stack:
```
docker compose up --build
```

Run the integration test from Question 2 against the Compose stack and check the result in the dashboard. Confirm that the app stores its data in PostgreSQL.

Which hostname should the API use to connect to the postgres service in Docker Compose?
- localhost
- postgres ✅
- host.docker.internal
- 0.0.0.0

#### SOLUTION

Q4 — PostgreSQL + Docker Compose

Now we move on to a more important step: replacing the SQLite database with PostgreSQL and running Agent Relay alongside PostgreSQL using Docker Compose.

The target architecture is as follows:
```text
┌──────────────────────┐
│     agent-relay          │
│      FastAPI             │
│       :8000              │
└──────────┬───────────┘
             │
         PostgreSQL
        postgres:5432
             │
┌──────────▼───────────┐
│     persistent           │
│       volume             │
└──────────────────────┘
```
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

---
## Question 5: Deploy to Kubernetes

Ask your coding agent to install kind and kubectl if needed, then create a local Kubernetes cluster.

Create manifests in k8s/ for Agent Relay and PostgreSQL, including Services, persistent DB storage, and readiness checks. Load your Docker image into kind and deploy the application.

Check that the pods are ready. Open the dashboard through port forwarding and verify the task flow from Question 2.

Which Kubernetes resource keeps the requested number of application replicas running and manages updates?
- Service
- ConfigMap
- Deployment ✅
- Secret

#### SOLUTION

A Deployment maintains the number of application Pods according to the requested replica count and manages the application version update or rollout process.

In practice, we will follow this sequence:
```text
Docker image
    │
    ▼
kind cluster
    │
    ├── PostgreSQL
    │    ├── PersistentVolumeClaim
    │    ├── Deployment
    │    └── Service
    │
    └── Agent Relay
         ├── Deployment
         ├── Service
         └── readiness probe
              │
              ▼
         port-forward :8000
              │
              ▼
          Dashboard
              │
              ▼
       Q2 task flow
```

Since the homework instructions explicitly call for `kind` and `kubectl`, we start with a local Kubernetes environment.

1. Check if `kind` and `kubectl` are available

Run:
```bash
kind version
kubectl version --client
docker version
```

NOTES:

If either `kind` or `kubectl` is not available, do not install them manually yet. Send the output of these three commands.

After that, we will:
1. install missing components,
2. create the agent-relay cluster,
3. create K8s manifests,
4. load the local agent-relay image into kind,
5. deploy PostgreSQL + PVC,
6. deploy Agent Relay + readiness check,
7. ensure all Pods are Ready,
8. run kubectl port-forward for the dashboard,
9. rerun the Alice → Uppercase flow from Question 2.

The environment is ready. No need to install anything:
```text
• kind v0.27.0-alpha ✅
• kubectl v1.32.1 ✅
• Docker Engine 29.1.3 ✅
```
Now, create a local Kubernetes cluster.

1. Create a kind cluster

I recommend a specific cluster name for the homework:
```Bash
kind create cluster --name agent-relay
```

Once finished:
```Bash
kubectl cluster-info --context kind-agent-relay
```

Then:
```Bash
kubectl get nodes
```

Output:
```text
NAME                         STATUS   ROLES           AGE   VERSION
agent-relay-control-plane    Ready    control-plane   92s   v1.32.0
```

2. Ensure the correct context is being used
```Bash
kubectl config current-context
```

It should be:
```
kind-agent-relay
```

Then:
```Bash
kubectl get pods -A
```

You should see default Kubernetes pods such as coredns, etcd, kube-apiserver, etc.
```text
NAMESPACE            NAME                                                READY   STATUS    RESTARTS   AGE
kube-system          coredns-668d6bf9bc-52djr                            1/1     Running   0          5m26s
kube-system          coredns-668d6bf9bc-9sf5k                            1/1     Running   0          5m26s
kube-system          etcd-agent-relay-control-plane                      1/1     Running   0          5m30s
kube-system          kindnet-glwb5                                       1/1     Running   0          5m26s
kube-system          kube-apiserver-agent-relay-control-plane            1/1     Running   0          5m30s
kube-system          kube-controller-manager-agent-relay-control-plane   1/1     Running   0          5m30s
kube-system          kube-proxy-nvdqd                                    1/1     Running   0          5m26s
kube-system          kube-scheduler-agent-relay-control-plane            1/1     Running   0          5m34s
local-path-storage   local-path-provisioner-58cc7856b6-jwgf4             1/1     Running   0          5m25s
```

Let's move on to the main part of Q5: creating Kubernetes manifests for PostgreSQL + Agent Relay.

1. Create the k8s/ structure

From the repository:
```Bash
cd ~/projects/zoomcamp/myprojects/agent-relay
mkdir -p k8s
```

We will create:
```text
k8s/
├── postgres-secret.yaml
├── postgres-pvc.yaml
├── postgres-deployment.yaml
├── postgres-service.yaml
├── agent-relay-deployment.yaml
└── agent-relay-service.yaml
```

2. PostgreSQL Secrets
```Bash
cat > k8s/postgres-secret.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_DB: agent_relay
  POSTGRES_USER: agent_relay
  POSTGRES_PASSWORD: agent_relay_password
EOF
```

3. PostgreSQL persistent storage
```Bash
cat > k8s/postgres-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
specs:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

kind already has a local-path-provisioner, as seen from your output:
```
local-path-storage local-path-provisioner-... 1/1 Running
```

So this PVC should automatically get storage.

4. PostgreSQL Deployment

```
cat > k8s/postgres-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 10
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-pvc
EOF
```

5. PostgreSQL Service

This service is important because the Agent Relay will subsequently access the database using the hostname:
```text
posrgres
```

```
cat > k8s/postgres-service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
EOF
```

6. Agent Relay Deployment

Use the local image you successfully built earlier:
```text
agent-relay:local
```

Manifest:
```
cat > k8s/agent-relay-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-relay
spec:
  replicas: 1
  selector:
    matchLabels:
      app: agent-relay
  template:
    metadata:
      labels:
        app: agent-relay
    spec:
      containers:
        - name: agent-relay
          image: agent-relay:local
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          env:
            - name: RELAY_DATABASE_URL
              value: postgresql+psycopg://agent_relay:agent_relay_password@postgres:5432/agent_relay
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
EOF
```

7. Agent Relay Service

```Bash
cat > k8s/agent-relay-service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: agent-relay
spec:
  selector:
    app: agent-relay
  ports:
    - port: 8000
      targetPort: 8000
  type: ClusterIP
EOF
```

8. Ensure the files exist

Run:
```Bash
ls -la k8s/
```
You should see:
```text
agent-relay-deployment.yaml
agent-relay-service.yaml
postgres-deployment.yaml
postgres-pvc.yaml
postgres-secret.yaml
postgres-service.yaml
```

Let's move on to loading the image into Kind.

1. Load the `agent-relay:local` image into the cluster

Since the image was built on the Docker host, you need to explicitly load it into `kind`:
```Bash
kind load docker-image agent-relay:local --name agent-relay
```

Then verify:
```Bash
docker exec -it agent-relay-control-plane crictl images | grep agent-relay
```

You should see `agent-relay` with the `local` tag.
```text
docker.io/library/agent-relay                   local                224452bd13f26       371MB
```

2. Deploy all manifests

If the image has been entered, run:
```Bash
kubectl apply -f k8s/
```
```text
secret/postgres-secret created
persistentvolumeclaim/postgres-pvc created
deployment.apps/postgres created
service/postgres created
deployment.apps/agent-relay created
service/agent-relay created
```

3. Check resources

Run:
```Bash
kubectl get pods -o wide
```

then:
```Bash
kubectl get svc
kubectl get pvc
```

```text
(base) dataeng@linuxmint-vm:~/projects/zoomcamp/myprojects/agent-relay$ kubectl get pods -o wide
NAME                           READY   STATUS    RESTARTS      AGE   IP           NODE                        NOMINATED NODE   READINESS GATES
agent-relay-7b69996c7f-ft47l   1/1     Running   7 (10m ago)   17m   10.244.0.5   agent-relay-control-plane   <none>           <none>
postgres-d84f86fc6-pbr7f       1/1     Running   0             17m   10.244.0.7   agent-relay-control-plane   <none>           <none>
(base) dataeng@linuxmint-vm:~/projects/zoomcamp/myprojects/agent-relay$ kubectl get svc
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
agent-relay   ClusterIP   10.96.2.221    <none>        8000/TCP   17m
kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP    55m
postgres      ClusterIP   10.96.60.167   <none>        5432/TCP   17m
(base) dataeng@linuxmint-vm:~/projects/zoomcamp/myprojects/agent-relay$ kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgres-pvc   Bound    pvc-187951f3-29f3-4454-83eb-d015047aab1a   1Gi        RWO            standard       <unset>                 11m
```

4. Wait for the Agent Relay to be fully ready

Use:
```Bash
kubectl wait --for=condition=ready pod \
  -l app=agent-relay \
  --timeout=120s
```

If successful:
```Bash
pod/agent-relay-xxxxxxxxxx-xxxxx condition met
```

Then:
```Bash
kubectl get pods
```

Two main components:
```
                    Kubernetes / kind
                           │
             ┌─────────────┴─────────────┐
             │                           │
       PostgreSQL                    Agent Relay
             │                           │
     ┌───────┴───────┐                   │
     │               │                   │
   Secret           PVC                  │
     │               │                   │
     └───────┬───────┘                   │
             │                           │
        PostgreSQL Pod ◄────── Service ──┘
             │
          :5432

Agent Relay Pod
      │
   :8000
      │
Service agent-relay
      │
 port-forward
      │
localhost:8000
      │
   /health ✅
   /docs   ✅
```

Flows

```text
Docker image
    ↓
kind load docker-image
    ↓
Node (kind)
    ↓
Pod
    ↓
Container
    ↓
FastAPI :8000
    ↓
Service (ClusterIP)
    ↓
Endpoint
    ↓
Port-forward
    ↓
Browser / curl
```

Relay Agent last status:

```
| Komponen                      | Status          |
| ----------------------------- | ----------------|
| Image → kind                  | ✅ Functional  |
| kind → Pod                    | ✅             |
| Pod → FastAPI                 | ✅             |
| PostgreSQL Pod                | ✅             |
| PostgreSQL Service            | ✅             |
| Service → Agent Relay         | ✅             |
| Port-forward → Service        | ✅             |
| `/docs`                       | ✅             |
| `/health`                     | ✅             |
| Startup dependency PostgreSQL | ✅             |
| Current application           | ✅ Healthy     |

```

$ docker exec agent-relay-control-plane crictl images
```text
IMAGE                                           TAG                  IMAGE ID            SIZE
docker.io/kindest/kindnetd                      v20241212-9f82dd49   d300845f67aeb       39MB
docker.io/kindest/local-path-helper             v20241212-8ac705d0   baa0d31514ee5       3.08MB
docker.io/kindest/local-path-provisioner        v20241212-8ac705d0   04b7d0b91e7e5       22.5MB
docker.io/library/agent-relay                   local                224452bd13f26       371MB
docker.io/library/postgres                      16-alpine            81bd698b4594e       116MB
registry.k8s.io/coredns/coredns                 v1.11.3              c69fa2e9cbf5f       18.6MB
registry.k8s.io/etcd                            3.5.16-0             a9e7e6b294baf       57.7MB
registry.k8s.io/kube-apiserver-amd64            v1.32.0              73afaf82c9cc3       98MB
registry.k8s.io/kube-apiserver                  v1.32.0              73afaf82c9cc3       98MB
registry.k8s.io/kube-controller-manager-amd64   v1.32.0              f3548c6ff8a1e       90.8MB
registry.k8s.io/kube-controller-manager         v1.32.0              f3548c6ff8a1e       90.8MB
registry.k8s.io/kube-proxy-amd64                v1.32.0              aa194712e698a       95.3MB
registry.k8s.io/kube-proxy                      v1.32.0              aa194712e698a       95.3MB
registry.k8s.io/kube-scheduler-amd64            v1.32.0              faaacead470c4       70.6MB
registry.k8s.io/kube-scheduler                  v1.32.0              faaacead470c4       70.6MB
registry.k8s.io/pause                           3.10                 873ed75102791       320kB
```

---
### Question 6: CI/CD

Ask your coding agent to create .github/workflows/ci.yml that runs the starter's tests and your integration test against PostgreSQL, builds a new Docker image, and deploys it to your kind cluster only if the tests pass.

Run the workflow locally with act. Ask your agent to configure access to Docker and the kind cluster, including loading the new image into kind. Use a unique image tag for each version and wait for the rollout to finish.

Change the dashboard heading to Agent Relay v2 and run the workflow again. Verify that the tests pass and the new heading appears in the deployed dashboard.

What should happen if a test fails in this workflow?
- Deploy the new version and report the failure.
- Keep the existing version running and stop the deployment. ✅
- Delete the existing deployment.
- Deploy the previous image with the new tag.

#### SOLUTION

To complete the Q6 exercise, we need to create a workflow that:
```
Runs starter tests + integration tests.
Uses PostgreSQL for integration tests.
Builds a Docker image if all tests pass.
Assigns a unique tag to each build.
Loads the new image into kind.
Deploys/updates the Kubernetes Deployment.
Waits for the rollout to complete.
Tests the dashboard.
Changes the heading to "Agent Relay v2".
Runs the workflow again and verifies that the new version is actually deployed.
```

Check the repository status:
```Bash
cd ~/projects/zoomcamp/myprojects/agent-relay

git status --short
```
Then check if `act` is available:
```Bash
/usr/local/bin/act --version
```
And ensure the `kind` cluster is running:
```Bash
kubectl get nodes
kubectl get pods
```

Verify the current state of the application:

Since Q6 involves deploying to an already running cluster:
```Bash
kubectl get deployment
kubectl get svc
kubectl get pvc
```
And check the image currently in use:
```Bash
kubectl get deployment agent-relay \
-o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

After verifying `act`, we will create a workflow following this pattern:
```text
GitHub Actions / act
│
▼
PostgreSQL service
│
▼
pytest + integration test
│
PASS? 
│   │
NO   YES
│    │
│    ▼
│ Docker build
│    │
│    ▼
│ unique image tag
│    │
│    ▼
│ kind load docker-image
│    │
│    ▼
│ kubectl apply
│    │
│    ▼
│ rollout status
│    │
└────┴── failure stops here
```
Kondisi Kubernetes sudah siap

Resource Q5 sekarang terlihat benar:
```text
Deployment
├── agent-relay   1/1
└── postgres      1/1

Service
├── agent-relay   8000
└── postgres      5432

PVC
└── postgres-pvc  Bound
```

So, we can proceed to create the Q6 workflow.

Ensure the workflow directory exists:
```Bash
mkdir -p .github/workflows
```

The initial workflow is simple but meets the homework requirements:
```text
PostgreSQL runs as a service container.
pytest uses PostgreSQL.
The Docker image is built only after tests pass.
The image is tagged using the commit SHA.
The image is loaded into kind.
The Kubernetes Deployment is updated with that tag.
kubectl rollout status waiting for deployment to complete.
```

Make ci-cd.yml:

```
name: CI/CD

on: 
push: 
workflow_dispatch:

jobs: 
test: 
runs-on: ubuntu-latest 

services: 
postgres: 
image: postgres:16-alpine 
env: 
POSTGRES_USER: agent_relay 
POSTGRES_PASSWORD: test_password 
POSTGRES_DB: agent_relay 
ports: 
- 5432:5432 
options: >- 
--health-cmd "pg_isready -U agent_relay -d agent_relay" 
--health-interval 5s 
--health-timeout 5s 
--health-retries 10 

env: 
RELAY_DATABASE_URL: postgresql+psycopg://agent_relay:test_password@localhost:5432/agent_relay 

steps: 
- name: Checkout 
uses: actions/checkout@v4 

- name: Set up Python 
uses: actions/setup-python@v5 
with: 
python-version: "3.11" 

- name: Install uv 
uses: astral-sh/setup-uv@v5 

- name: Install dependencies 
run: uv sync --frozen 

- name: Run tests 
run: uv run pytest -q 

build-and-deploy: 
runs-on: ubuntu-latest 
needs: test 

env: 
IMAGE_NAME: agent-relay 
IMAGE_TAG: ${{ github.sha }} 
KIND_CLUSTER_NAME: kind 

steps: 
- name: Checkout 
uses: actions/checkout@v4 

- name: Build Docker image 
run: | 
docker build\ 
-t ${IMAGE_NAME}:${IMAGE_TAG} \ 
. 

- name: Install kind and kubectl 
run: | 
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64 
chmod +x ./kind 
sudo mv ./kind /usr/local/bin/kind 

curl -LO "https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl" 
chmod +x kubectl 
sudo mv kubectl /usr/local/bin/kubectl 

kind version 
kubectl version --client 

- name: Create kind cluster (if not exists) 
run: | 
if kind get clusters | grep -q "^${KIND_CLUSTER_NAME}$"; then 
if [ "$(docker inspect -f '{{.State.Running}}' ${KIND_CLUSTER_NAME}-control-plane 2>/dev/null)" = "true" ]; then 
echo "Cluster '${KIND_CLUSTER_NAME}' already exists and is running, skip creation." 
else 
echo "Cluster '${KIND_CLUSTER_NAME}' is registered but the container is not running, delete it then recreate it..." 
kind delete cluster --name ${KIND_CLUSTER_NAME} 
kind create cluster --name ${KIND_CLUSTER_NAME} 
fi 
else 
kind create cluster --name ${KIND_CLUSTER_NAME} 
fi 

- name: Export kubeconfig 
run: | 
kind export kubeconfig --name ${KIND_CLUSTER_NAME} 

- name: Load image into kind 
run: | 
kind load docker-image\ 
${IMAGE_NAME}:${IMAGE_TAG} \ 
--name ${KIND_CLUSTER_NAME} 

- name: Apply Kubernetes manifests 
run: | 
kubectl apply -f k8s/ 

- name: Update Kubernetes deployment 
run: | 
kubectl set image deployment/agent-relay \ 
agent-relay=${IMAGE_NAME}:${IMAGE_TAG} 

- name: Wait for rollout 
run: | 
kubectl rollout status deployment/agent-relay\ 
--timeout=120s 

- name: Test dashboard 
run: | 
kubectl port-forward \
service/agent-relay \
18000:8000 > /tmp/port-forward.log 2>&1 &

PF_PID=$! 
trap 'kill $PF_PID' EXIT

for i in {1..30}; do
if curl -fsS http://127.0.0.1:18000/ > /tmp/dashboard.html; then
break
fi
sleep 2
done

grep -q "Agent Relay v2" /tmp/dashboard.html

echo "Dashboard test passed"
```

docker-compose.yml
```Bash
services:
  postgres:
    image: postgres:16-alpine
    container_name: agent-relay-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
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
      RELAY_DATABASE_URL: postgresql+psycopg://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8000:8000"

volumes:
  postgres_data:
```

Re-buidl agent-relay-api container with host port 18000:
```Bash
docker compose down
docker compose up -d --build
```

Open browser: http://127.0.0.1:18000/

Run the test job first

We don't need to run the deployment immediately. Run only the test job:
```Bash
act push -j test \
-P ubuntu-latest=catthehacker/ubuntu:act-latest
```
This tests the most critical parts first:
```text
PostgreSQL
↓
RELAY_DATABASE_URL
↓
uv sync
↓
pytest
```

If successful, the final output should show:
```text
✓ test
```

and pytest should show:
```text
5 passed
```

After the test succeeds

Then run the full workflow with host access:
```Bash
act push \
-P ubuntu-latest=catthehacker/ubuntu:act-latest \
--container-options "-v /var/run/docker.sock:/var/run/docker.sock -v $HOME/.kube:/root/.kube:ro --network host"
```

then:
```Bash
act -j build-and-deploy
```
