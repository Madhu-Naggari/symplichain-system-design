# Symplichain Hackathon Submission

This repository contains my submission for the Symplichain Software Engineering Intern hackathon.

- Written response: `submission.pdf`
- CI/CD workflow: `.github/workflows/deploy.yml`

## Part 1: Shared Gateway Problem

### Architecture

I would keep the design simple:

- `API gateway (Django/FastAPI)`: receives requests from all customers
- `Redis`: stores per-customer queues and scheduler state
- `Celery`: runs the dispatcher and retry jobs
- `PostgreSQL`: stores request status and audit trail
- `External API`: limited to 3 requests/second

Why this approach:

- The main challenge is not computation, it is outbound traffic control
- Redis + Celery is enough for this scale and much cheaper than introducing Kafka
- Keeping the rate limiter in one place makes the system easier to reason about

### How I Would Enforce the 3 req/sec Limit

I would not allow every worker to call the external API directly.

- Each customer gets a Redis queue: `queue:<customer_id>`
- The gateway validates the request quickly, stores its metadata, and pushes it to that queue
- A single dispatcher runs once every second
- On each tick, it selects at most 3 requests total and sends them to the external API

Why this guarantees the limit:

- Only the dispatcher is allowed to make outbound calls
- The dispatcher has a hard cap of 3 sends per second
- Any excess traffic simply waits in Redis

If I wanted high availability later, I would run a standby dispatcher and protect the active one with a Redis lock.

### Fairness Across Customers

A single global FIFO queue is simple, but not fair. One heavy customer could fill the queue and delay everyone else.

To avoid that, I would use **per-customer queues with round-robin scheduling**:

- Maintain one queue per customer
- Keep a rotating pointer to the next customer to serve
- In each 1-second cycle, take at most 1 request from a customer before moving to the next one
- Continue until 3 requests are selected

This avoids starvation. A noisy customer can still use spare capacity, but not at the cost of blocking everyone else.

### Retry Strategy

Retries should be selective:

- `Retryable`: timeout, temporary 5xx, connection reset, occasional 429
- `Non-retryable`: invalid payload, auth failure, bad request

Policy:

- Exponential backoff with jitter
- Example delays: 2s, 5s, 15s, 30s, 60s
- Max 5 attempts
- After that, mark the request as `failed` and surface it for review

Why:

- Backoff prevents hammering a slow dependency
- Jitter avoids synchronized retries
- Not retrying 4xx errors keeps the queue healthy

### Request Flow

1. Customer sends request to the shared gateway.
2. Gateway authenticates, validates, stores request metadata, and pushes the job into that customer’s Redis queue.
3. Dispatcher wakes up every second.
4. Dispatcher selects up to 3 jobs using round-robin across non-empty queues.
5. Dispatcher sends requests to the external API.
6. Success updates the request state.
7. Retryable failures are rescheduled with backoff.
8. Permanent failures are marked as failed.

### Trade-offs

- `Why this approach:` simple, cheap, and easy to defend in production
- `Why not a global queue:` easier, but unfair during tenant bursts
- `Why not Kafka:` useful at larger scale, but too heavy for this problem
- `Why not worker-side throttling:` harder to guarantee a strict global limit across multiple workers

## Part 2: Mobile App Architecture

### Interaction Model

I would choose a **hybrid model**:

- Main workflows stay click-based and predictable
- AI acts as an assistant, not the primary interface

Examples:

- Click-based flows for order creation, approval, tracking, and image upload
- AI assistance for search, summarization, and suggested next actions

Why:

- Operations users usually prefer speed and reliability over chat-first UX
- A fully AI-driven app is harder to test and less predictable
- Hybrid gives AI value without making the product feel fragile

### Tech Stack Choice

I would choose **React Native with TypeScript**.

Why:

- One codebase for iOS and Android means faster delivery
- Lower cost than maintaining separate Swift and Kotlin apps
- Better maintainability if the web team already uses React
- Easier to share API patterns and UI decisions with the web app

I would only choose native apps first if offline support, camera/scanner features, or platform-specific SDKs were a major requirement.

## Part 3: CI/CD Pipeline

I would keep deployment branch-driven:

- `staging` branch deploys to staging
- `main` branch deploys to production

The actual workflow is included in [`.github/workflows/deploy.yml`](./.github/workflows/deploy.yml).

High-level pipeline:

- Run backend tests for Django
- Build the React frontend
- Deploy backend to EC2 over SSH
- Sync frontend build output to S3

Why this is the right level of complexity:

- It is simple enough for a small team to maintain
- It matches the current stack directly
- It keeps staging and production separate without adding unnecessary tooling

Later improvements I would suggest:

- Dockerize backend deployment for consistency
- Use Terraform for infra management
- Replace long-lived AWS keys with GitHub OIDC + IAM roles
- Add CloudFront invalidation after S3 deploys

## Part 4: Debugging Scenario

Problem flow:

`Driver App -> Django API -> S3 -> Celery -> Bedrock -> PostgreSQL`

I would debug in the same order as the request path instead of jumping straight to Celery or Bedrock.

### Step-by-step Approach

1. Reproduce the issue with one known image and one known user.
   Why: it is much easier to trace one request end to end than to reason from noisy production traffic.

2. Check whether the request reaches Django.
   Why: if the API never receives it, the issue is in the client, auth, or network path.
   Tools: Django logs, nginx/ALB logs, CloudWatch.

3. Verify the S3 upload step.
   Why: if the object never lands in S3, the bug is before Celery and Bedrock.
   Tools: S3 console, boto3 error logs, CloudWatch.

4. Confirm Celery task creation and worker pickup.
   Why: if S3 succeeded but no task was queued, the issue is in app logic or broker flow.
   Tools: Celery worker logs, Flower, Redis queue inspection.

5. Inspect the Bedrock call.
   Why: sometimes the upload succeeds and the actual failure is in downstream inference.
   Tools: CloudWatch, worker logs, Bedrock invocation errors.

6. Check PostgreSQL writeback.
   Why: the system may look like an upload failure when the final DB update is the real issue.
   Tools: Django logs, PostgreSQL logs, targeted SQL checks.

7. Compare one failing case with one successful case.
   Why: patterns like file type, file size, or environment mismatch often show up quickly.

8. Add temporary tracing across each step if the issue is intermittent.
   Why: step-level visibility reduces guesswork.

## Conclusion

I focused on simple, scalable, and cost-efficient solutions that are realistic for a small engineering team to build and operate. The common theme across all four parts is to keep the design easy to reason about, make failure handling explicit, and avoid over-engineering too early.
