# Lost or Run Twice

What Python job queues lose or run twice when a worker dies mid-task.

**Status: in progress. There are no results yet.** This README states the question, the method and each library's documented promise before any measurement, so the results cannot be fitted to a story afterward.

## The question

When a worker dies in the middle of a task, or after the task's side effect but before it is acknowledged, or the broker restarts, or the connection drops, what happens to the task? It can be lost, run twice, left stuck, or redelivered. If it is redelivered, how long does that take? And does the measured behavior match what the library's own docs promise?

## Why it matters

Most AI backends run their LLM calls, document processing and agent steps as background tasks. A task that runs twice can send an email twice, write a record twice, or pay for the same LLM call twice. A task that is lost silently drops someone's work. Teams usually pick a queue for its features and never see a failure matrix.

## What each library promises

From each library's own docs, read on 2026-10-18.

| Library | What happens when a worker is killed mid-task | Source |
|---|---|---|
| Celery | By default a task is acknowledged before it runs: "Late ack means the task messages will be acknowledged after the task has been executed, not right before (the default behavior)." With `task_reject_on_worker_lost` off (the default), "the worker will acknowledge tasks when the worker process executing them abruptly exits or is signaled." Turning on late acks plus reject-on-worker-lost re-queues the task, and the docs warn that such tasks "must be idempotent as they may be executed more than once" and that reject-on-worker-lost "can cause message loops." With Redis, tasks of a forcefully terminated worker "will not be consumed again until the Visibility Timeout has passed," which defaults to one hour. | [Configuration](https://docs.celeryq.dev/en/stable/userguide/configuration.html), [Redis broker](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/redis.html) |
| RQ | A job whose worker "is terminated before it finished that job" is collected by another worker's maintenance task, which "moves it to the FailedJobRegistry." Retries are opt-in through `Retry(max, interval)`. The docs do not say whether an abandoned job is retried. | [Exceptions and retries](https://python-rq.org/docs/exceptions/) |
| Dramatiq | "Tasks are only ever acked when they're done processing." The page notes it was last updated in 2019. | [Motivation](https://dramatiq.io/motivation.html) |
| Procrastinate | A `SIGKILL` can leave jobs in the `doing` state, and without action they "will remain in the queue forever." Workers send a heartbeat every 10 seconds and count as stalled after 30. Retrying stalled jobs needs a periodic task you add yourself, calling `get_stalled_jobs()` and `retry_job()`. | [Retry stalled jobs](https://procrastinate.readthedocs.io/en/stable/howto/production/retry_stalled_jobs.html) |
| Hatchet | Tasks are durable: "Hatchet persists it, its state, and its results even after it finishes running." Retries are a per-task setting. The tasks page does not say what happens when a worker crashes. Durable tasks resume from a checkpoint log, described as exactly-once for completed steps. | [Tasks](https://docs.hatchet.run/v1/tasks), [Durable execution](https://docs.hatchet.run/v1/durable-execution) |

Where the docs say nothing, the matrix supplies the answer.

## The matrix

| Library | Broker | Configurations |
|---|---|---|
| Celery | Redis, RabbitMQ | default; late acks plus reject-on-worker-lost |
| RQ | Redis | default; with `Retry` |
| Dramatiq | Redis, RabbitMQ | default |
| Procrastinate | Postgres | default; with the stalled-job retry task |
| Hatchet | Hatchet engine on Postgres | no retries; 3 retries |

Faults:

1. Kill the process running the task, mid-task (`SIGKILL`)
2. Kill the whole worker, mid-task (`SIGKILL`)
3. Kill after the task's side effect is written but before it is acknowledged
4. Restart the broker or Postgres mid-task
5. Drop the worker's connection to the broker, through [Toxiproxy](https://github.com/Shopify/toxiproxy)

Stretch: clock skew, and a full disk on the broker.

## What gets measured

Every task carries an idempotency key. Every attempt writes its start and its side effect to an append-only ledger in a separate Postgres that no fault touches.

- **Lost:** enqueued, but its side effect never appears within the observation window.
- **Run twice:** the side effect appears more than once for the same key.
- **Stuck:** still claimed at the end of the window, with no live worker holding it.
- **Time to redelivery:** from the fault to the next attempt starting, reported as p50 and p99.

The observation window is longer than each library's documented redelivery timeout at its configured value, and each cell lists the timeout values it ran with.

## Method

- Each library pinned to one version in `versions.lock`, configured as its docs describe for each configuration.
- Faults fire at fixed points: the task signals the harness when it reaches the point, and the harness sends the fault, so timing is repeatable.
- At least 3 runs per cell, each with a fixed number of tasks.
- Published numbers come from one named Linux cloud VM. The instance type, kernel and Docker versions are recorded with the results.

## Results

Not yet. This section fills in once every cell has at least 3 runs. Raw ledgers go in `data/`.

## Reproduce

One command will rebuild the whole matrix from `versions.lock`, and one will run a single cell. Both arrive with the harness.

## Prior art

[Jepsen](https://jepsen.io/analyses) tests brokers and databases under network partitions, including [RabbitMQ](https://aphyr.com/posts/315-jepsen-rabbitmq). Jepsen tests the broker or database cluster itself. This project tests the task library on top of a single broker, at task boundaries, the way an application team runs it, and puts the libraries side by side under the same faults and the same ledger.

## Upstream

None yet. Every case where measured behavior contradicts a library's documented promise gets a minimal reproducer filed with that library, and a fix or docs PR where possible. Links will go here.

## Limitations

- One host and single-node brokers. Partitions between broker nodes are Jepsen's territory and out of scope.
- Results hold for the pinned versions and configurations; defaults change.
- Synthetic tasks. Real workloads have different durations and side effects.

## License

Code is licensed under [Apache-2.0](LICENSE). Data in `data/` is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

## AI assistance

AI coding tools help with scaffolding. Every measurement, ledger check and conclusion is reviewed by hand.
