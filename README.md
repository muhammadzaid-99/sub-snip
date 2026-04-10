# SubSnip

SubSnip is a distributed system for pulling hardcoded subtitles out of videos with OCR.

Send a video URL plus what to read (language, time range, and the screen region where the
subtitles sit) and you get a task ID back. Workers pick the job off a queue, download the
video, and hand the selected frames to an OCR pass. More throughput is a replica count away.

Go for the API and job handling, Python for download and OCR through videocr and PaddleOCR.

![Architecture](docs/architecture.png)

## Features

- **Targeted extraction.** Language, start and end time, frame skip interval, and a subtitle
  bounding box. OCR reads one strip of sampled frames instead of every full frame.
- **Queue backed.** The API only validates and publishes. Workers consume independently, so
  a slow job blocks nothing else.
- **Horizontally scalable.** Stateless workers compete on one queue. No worker knows how
  many peers exist.
- **Status tracking.** Each task gets an ID with a state you can poll over HTTP.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/submit-task` | Validate and enqueue a job, returns `task_id` |
| GET | `/task-status/{taskID}` | Look up the recorded state of a task |

```json
{
  "video_url": "https://www.youtube.com/watch?v=...",
  "language_code": "ch",
  "start_time": "00:01:30",
  "end_time": "00:04:00",
  "frames_to_skip": 4,
  "subtitle_box": {
    "top_left":     { "x": 200, "y": 800 },
    "bottom_right": { "x": 1700, "y": 1000 }
  }
}
```

Missing fields or an all-zero box are rejected, so a bad job never reaches a worker.

## How it works

**Go to Python bridge.** The orchestration is Go, the OCR tooling is Python. The worker
starts the script with `exec.CommandContext` and pipes the task in as JSON on stdin, while
two goroutines drain stdout and stderr into the Go log.

- Nested values like the bounding box need no shell escaping, and new fields are a struct
  change rather than an argument convention change.
- Download and OCR progress shows up live instead of buffering until exit.

**Every job is bounded in time.** OCR on video is open ended, and a stalled job otherwise
holds a worker forever. Each run gets a five minute deadline on a context derived from the
worker's shutdown context.

- `CommandContext` kills the Python process on expiry instead of abandoning it.
- `ctx.Err()` is checked for `DeadlineExceeded`, so a timeout reads differently from a
  script that exited with an error.
- The same mechanism tears down a running job when the container is asked to stop.

**Acknowledgements are manual.** Auto ack drops jobs when a worker crashes; blind requeue
replays a doomed message forever at the cost of a full download each time. So the consumer
runs with auto ack off and splits the failure cases.

- Unparseable bodies are nacked with requeue disabled.
- Anything that got as far as running is acked once it finishes, pass or fail.
- Result: a dead worker releases its message, poison jobs leave the queue.

**One shared queue package, fatal on startup.** Both binaries run the same `init`: read
`RABBITMQ_URL` with a localhost fallback, open a channel, declare the queue durable.

- Declaration is idempotent, so whichever process starts first creates it.
- Any failure is fatal, so a misconfigured container exits visibly instead of dropping jobs.
  Compose restarts it once the broker is up.

**Graceful shutdown.** Both binaries root their context in `signal.NotifyContext` for SIGINT
and SIGTERM. The producer serves HTTP on a goroutine, then on signal closes AMQP and calls
`srv.Shutdown` with a five second window. The worker's consume loop selects on the same
context.

**Two runtimes, one image.** The worker Dockerfile builds in three stages: system deps plus
Python requirements, Go compile on Alpine, then a clean Python base that copies in the
prepared Python tree, the binary, and `scripts/` as its own last layer.

- Editing the script or the Go code reuses the cached dependency layer.
- Neither build toolchain ends up in the runtime image.

## Running it

```bash
docker compose up --build
```

RabbitMQ management on 15672, producer API on 8080, one worker. For a pool, uncomment the
`deploy` block in `docker-compose.yml`:

```yaml
worker:
  deploy:
    replicas: 3
```

Submit and poll:

```bash
curl -X POST http://localhost:8080/submit-task \
  -H "Content-Type: application/json" -d @task.json

curl http://localhost:8080/task-status/task-<uuid>
```

## Configuration

| Setting | Where | Notes |
| --- | --- | --- |
| PaddlePaddle build | `scripts/requirements.txt` | A CUDA build and a CPU build on different indexes. Enable exactly one. This is the setting that matters most, since it decides whether OCR runs on the GPU |
| `RABBITMQ_URL` | Environment | Defaults to `amqp://guest:guest@localhost:5672/`, which is what makes running outside Compose work |
| `replicas` | `docker-compose.yml` | Size of the worker pool |
| Job timeout | `cmd/worker/main.go` | Five minutes per task |

## Project layout

| Area | Path | What lives there |
| --- | --- | --- |
| Entry points | `cmd/producer`, `cmd/worker` | API server and queue consumer, each with its own Dockerfile |
| HTTP layer | `internal/api` | Decoding, validation, task ID assignment, status lookup |
| Messaging | `internal/queue` | Connection lifecycle, durable queue declaration, publishing |
| Shared types | `internal/models` | Task schema used by both binaries and the Python script |
| State | `internal/status` | Task state store behind a read write mutex |
| Extraction | `scripts` | Python download and OCR stage plus its dependency list |
| Deployment | `docker-compose.yml` | Broker, producer, and worker wired together |

## Requirements

Docker and Docker Compose run the whole stack. For local development you need Go 1.24 and
Python 3.11, plus ffmpeg and the OpenCV system libraries.

## OCR engine

[videocr-PaddleOCR](https://github.com/oliverfei/videocr-PaddleOCR), built on PaddleOCR for
reading burned in subtitles from video frames. In the current script the download stage runs
and the videocr call is held commented out until PaddlePaddle is pinned to the target device.
