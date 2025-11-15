
<p align="center">
  <img src="job_scheduler.svg" height="800" alt="Job Scheduler"/>
</p>

**Step 0 — Job Submission**

Job consumer receives a new job request from Queue. Computes the scheduled_at time for a job and
Inserts into `job_type` and `job_run` tables

```
INSERT INTO job_type (name, schedule_cron, artifact_id, entrypoint)
VALUES ('customer_sync', '0 2 * * *', 'artifact-123', 'main:run');

INSERT INTO job_run (job_type_id, status, scheduled_at)
VALUES (1, 'queued', '2025-11-17T02:00:00Z');

```

✅ This represents the job template.

**Step 2 — Scheduler Tick / Polling**

Scheduler runs every few seconds:

```
SELECT *
FROM job_run
WHERE status = 'queued'
  AND scheduled_at <= NOW() + INTERVAL 'poll_threshold'
ORDER BY scheduled_at ASC
LIMIT 100;
```

Enqueues jobs into the Distributed Priority Queue (DPQ).
Updates job_run.status → enqueued.
Only the scheduler touches the DB; workers are decoupled.

**Step 3 — Controller / Dispatcher**

* The controller pulls jobs from Message Queue and assigns job to workers.
* Updates job_run.status in DB:
	* running when worker starts
	* success / failed when worker finishes
* On failure, controller can:
	* Re-enqueue the same job_run
	* Or create a new job_run with status='queued' and scheduled_at = NOW() + retry_interval
* Dead-letter jobs after exceeding max retries.
* If a recurring job finished executing, the Controller sends an event into message queue for the scheduler so that it can create a new row in `job_run` table 

** Step 4- Worker **

Once a worker is assigned a job, it can pull artiface from the object storage and executed the artifact
and reports the job status to the controller so that controller can update the job run_table accordingly.

** Step 5 - Recurring Job Rescheduling **

After a recurring job runs successfully, the scheduler received an event from Message queue, and it calculates next execution from `job_type.schedule_cron` and Inserts new job_run row:

```
INSERT INTO job_run (job_type_id, status, scheduled_at)
VALUES (1, 'queued', '2025-11-18T02:00:00Z');
```

This ensures a rolling window of scheduled executions.


---

## Running Multiple Schedulers

Each scheduler instance is pinned to a shard.

Scheduler polls only its shard for due job_run rows.

Precomputes job_run rows for new job_type entries in its shard.

Enqueues jobs to the DPQ as usual.