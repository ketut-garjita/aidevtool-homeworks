# Homework 3: Test, Containerize, and Deploy an AI-Assisted App

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

## Questions

Answer the questions below to complete your homework.

### 1. Which description matches the project's architecture? (1 point)

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
### 2. Which task status does the sender see after the recipient submits its result? (1 point)

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

### Answer to Question 2: completed

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
....                                                                                                                                                           [100%]
========================================================================== warnings summary ===========================================================================
.venv/lib/python3.11/site-packages/fastapi/testclient.py:1
  /home/dataeng/projects/zoomcamp/myprojects/agent-relay/.venv/lib/python3.11/site-packages/fastapi/testclient.py:1: StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.
    from starlette.testclient import TestClient as TestClient  # noqa

.venv/lib/python3.11/site-packages/starlette/testclient.py:53
  /home/dataeng/projects/zoomcamp/myprojects/agent-relay/.venv/lib/python3.11/site-packages/starlette/testclient.py:53: DeprecationWarning: The anyio.abc.BlockingPortal alias is deprecated, use anyio.from_thread.BlockingPortal instead.
    _PortalFactoryType = Callable[[], AbstractContextManager[anyio.abc.BlockingPortal]]

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
5 passed, 2 warnings in 4.38s

Question 2 is complete:

✅ Two agents successfully registered.
✅ Sender created a task via HTTP API.
✅ Recipient claimed the task via HTTP API.
✅ Recipient submitted the result via HTTP API.
✅ Sender can see the task as completed.
✅ "HELLO AGENT RELAY" output is stored.
✅ Acceptance flow automated as an integration test.
✅ Test uses the API and an actual SQLite database for the test environment.

Question 2 status: completed.

          
### 3. Which Docker option publishes a container's port to your machine? (1 point)
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
| docker build | ✅ Success |
| Image agent-relay:local | ✅ 361 MB |
| Container port 8000 published | ✅ |
| GET /health | ✅ {"status":"ok"} |
| GET /ready | ✅ {"status":"ready"} |

So the flow is proven:
