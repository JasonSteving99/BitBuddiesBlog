---
title: "Building BitBuddies: How Temporal Turned a Half Dozen (and Counting) AI Models Into a Reliable Animation Pipeline"
date: 2026-01-16T11:00:00-00:00
draft: false
tags: ["temporal", "ai", "orchestration", "python", "workflow"]
categories: ["engineering", "backend"]
author: "BitBuddies Team"
description: "Learn how Temporal's durable execution model orchestrates multiple AI models, handles failures gracefully, and ensures users never get double-charged."
cover:
    image: ""
    alt: "BitBuddies Animation Pipeline"
    relative: false
---

## The Problem: Orchestrating Chaos

Here's what happens when someone uploads a photo to [BitBuddies](https://bitbuddies.app): we need to wrangle a half dozen (and counting) different AI models in sequence, each with their own quirks, rate limits, and failure modes, to transform that image into an animated transparent character. Oh, and we need to charge them money *exactly once*, never lose their place if something crashes, and text me immediately if anything goes sideways.

No big deal, right?

Temporal is the perfect tool for this kind of challenge. The pipeline orchestrates a sequence of AI models: one filters NSFW content, another analyzes orientation and validates outputs, a third generates cartoon versions (sometimes twice for full-body expansion), another creates segmentation masks for both images and video, yet another generates 5-second animations, and finally ffmpeg transcodes and streams gigabytes directly to cloud storage.

The beauty of Temporal is how it handles all this complexity while ensuring users get automatic refunds if anything fails, and I get alerted on my phone with enough context to actually fix the problem.

Temporal makes these guarantees straightforward to implement and reason about.

## The Core Pattern: Durable Promises

One of Temporal's most powerful features is that workflows aren't just state machines, they're **durable promises**. When I write:

```python
# Charge the user first
self.charged_token_unique_id = await workflow.execute_activity(
    charge_bit_buddy_token,
    ChargeTokenInput(...),
    retry_policy=RetryPolicy(maximum_attempts=3),
    start_to_close_timeout=timedelta(minutes=1)
)
```

That token charge becomes *permanent history*. If my worker crashes, restarts, or gets abducted by aliens, Temporal remembers: "You charged this user. Act accordingly."

This enabled the holy grail: **automatic refunds wrapped around the entire workflow**.

```python
@workflow.run
async def run_with_alert_on_failure(self, input_data):
    try:
        await self.run(input_data)
    except Exception as err:
        if self.charged_token_unique_id:
            # Refund with idempotency - never double-refund
            await workflow.execute_activity(
                refund_bit_buddy_token,
                RefundBitBuddyTokenInput(
                    token_unique_id=self.charged_token_unique_id,
                    idempotency_key=workflow.uuid4(),
                    refund_reason=f"Workflow failed: {str(err)}"
                ),
                retry_policy=RetryPolicy(maximum_attempts=3)
            )

        # Alert me with full context
        await workflow.execute_activity(
            send_alert_activity,
            SendAlertInput(
                message=f"Workflow {workflow_info.workflow_id} failed...",
                url=f"https://temporal.ui/workflows/{workflow_info.workflow_id}"
            )
        )
        raise  # Re-raise to mark workflow as failed
```

Now when something breaks at 2 AM, I wake up to a Pushover notification with a direct link to the Temporal UI showing *exactly* which activity failed, what the error was, and confirmation that the user was refunded. No frantic queries trying to figure out if someone got double-charged.

## The Technical Gnarly Bits

### Challenge 1: Don't Re-run Expensive AI Calls

When you submit a video generation job to FAL AI, it costs money and takes 2-3 minutes. If my worker crashes mid-polling, I *absolutely cannot* resubmit that job. This is one of those problems that keeps you up at night in a traditional system: how do you maintain the connection to a long-running external job when your process might die at any moment?

Temporal makes this elegant with a three-activity pattern:

```python
# Activity 1: Submit (idempotent, returns job_id)
job_id = await workflow.execute_activity(submit_fal_ai_job_activity, ...)

# Activity 2: Poll with heartbeats (infinite retries!)
await workflow.execute_activity(
    poll_fal_ai_job_activity,
    PollInput(job_id=job_id),
    retry_policy=RetryPolicy(maximum_attempts=0),  # Retry forever
    heartbeat_timeout=timedelta(minutes=2),  # But detect stuck workers
    start_to_close_timeout=timedelta(minutes=10)
)

# Activity 3: Fetch result
result = await workflow.execute_activity(get_fal_ai_job_result_activity, ...)
```

**The magic:** Heartbeat timeout means if a worker dies mid-poll, Temporal reschedules the polling on a healthy worker. The expensive AI job keeps running; we just resume waiting for it.

Without Temporal, you'd need to build a separate job tracking system with a database to store job IDs, implement your own polling logic with failure recovery, and handle worker crashes gracefully. With Temporal, it's just three activities with different retry policies. The workflow's durable execution history means the job_id persists across worker restarts automatically.

### Challenge 2: Rate Limiting Across Concurrent Workflows

When *many* users hit "Generate" simultaneously (think product launch or viral traffic), we need to avoid slamming FAL AI with too many concurrent requests. But workflows are isolated, so how do you share state?

Enter the globally-cached semaphore:

```python
@lru_cache(maxsize=None)
def get_globally_cached_semaphore(*, name, max_concurrency):
    return Semaphore(max_concurrency)

# In workflow:
async with self.fal_semaphore:
    # Only N workflows can be in this block simultaneously
    job_id = await submit_fal_ai_job(...)
```

Python's `lru_cache` ensures all workflow instances share the *same* semaphore object. Simple, effective, and it feels like cheating.

**Here's what makes this powerful:** This single semaphore lives on one machine running a Temporal worker, but Temporal's distributed execution model means the actual *activities* can execute on any number of machines. The workflow orchestration (the Python function, state, semaphore) runs on one worker, while the activity executions (submitting jobs, polling, downloading results) can be distributed across dozens or hundreds of activity workers. Temporal's fine-grained placement capabilities let you scale workflow orchestration separately from activity execution, so this simple semaphore pattern coordinates work that might be happening across an entire cluster.

### Challenge 3: Deterministic Idempotency Keys with workflow.uuid4()

One of Temporal's most underrated features is `workflow.uuid4()`. Unlike regular `uuid.uuid4()`, which generates a different value on every call, `workflow.uuid4()` is **deterministic within a workflow execution**. Call it twice in the same workflow run, you get the same UUID. Replay the workflow from history, you get the same UUID.

This is transformative for idempotency:

```python
# Activity that charges a token
await workflow.execute_activity(
    charge_bit_buddy_token,
    ChargeTokenInput(
        user_id=input_data.user_id,
        token_unique_id=input_data.token_unique_id,
        idempotency_key=workflow.uuid4(),  # Stable across retries!
    ),
    retry_policy=RetryPolicy(maximum_attempts=3)
)

# Activity that refunds if workflow fails
await workflow.execute_activity(
    refund_bit_buddy_token,
    RefundBitBuddyTokenInput(
        token_unique_id=self.charged_token_unique_id,
        idempotency_key=workflow.uuid4(),  # Different UUID than above, but stable
        refund_reason=f"Workflow failed: {str(err)}"
    ),
    retry_policy=RetryPolicy(maximum_attempts=3)
)
```

**Why this matters:** If the charge activity retries due to a transient failure, it will use the *exact same* idempotency key on retry. My billing system can safely deduplicate without ever double-charging a user. And if the refund activity retries, it uses its own stable idempotency key, preventing double-refunds.

Every `workflow.uuid4()` call gets a deterministic UUID based on its position in the workflow execution history. It's one of those features that seems small until you realize how much complexity it eliminates.

### Challenge 4: Real-time Progress Updates

Users want to see progress: "Generating cartoon... Creating mask... Animating..."

For this use case, I went with a hybrid approach that combines Temporal's workflow context with an external pub/sub system:

```python
# From workflow activity:
await workflow.execute_activity(
    publish_progress_update,
    ProgressUpdate(workflow_id=workflow.info().workflow_id, step="Generating video"),
    retry_policy=RetryPolicy(maximum_attempts=1)  # Don't block workflow if publish fails
)

# FastAPI server maintains SSE connections:
@app.get("/workflow/{workflow_id}/stream")
async def stream_updates(workflow_id: str):
    async def event_generator():
        async for event in subscribe_to_workflow(workflow_id):
            yield f"data: {json.dumps(event)}\n\n"
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

Workers HTTP POST to the API server, which fans out via Server-Sent Events to browsers. Simple, scalable, and we keep event history so late-joining clients can catch up.

## Key Takeaways for Your Temporal Projects

1. **Split expensive operations into submit/poll/fetch** - Worker crashes shouldn't resubmit expensive jobs. Use heartbeat timeouts to detect stuck polling.
2. **Use workflow.uuid4() for idempotency keys** - Deterministic UUIDs across retries eliminate an entire class of double-charging bugs.
3. **Different retry policies for different activities** - Polling can retry forever with heartbeats. Billing should fail fast with limited attempts.
4. **Wrapper workflows for cross-cutting concerns** - My `run_with_alert_on_failure` wrapper adds refunds and alerting to any workflow without duplicating code.
5. **LRU cache for shared resources** - Global semaphores for rate limiting work beautifully when you need to coordinate across workflow instances.
6. **Hybrid approaches for non-critical side effects** - Progress updates can go through external pub/sub while critical operations stay in the workflow.
7. **Leverage durable execution history** - Job IDs, state, and idempotency keys persist automatically across worker restarts.

## What's Next: Physical Stickers!

The next big feature is **physical sticker generation**. Soon you'll be able to order physical stickers of your Bit Buddy and slap them on your laptop, water bottle, or Temporal conference badge. The best part? Each sticker includes a QR code that, when scanned, brings your Bit Buddy to life with its full animation.

Temporal is going to make this delightfully straightforward: long-running workflows can track order fulfillment, signal updates when stickers ship, and automatically handle refunds if printing fails. The foundation is *already there*.

## Try It Yourself

Want to see this in action? Head to **[bitbuddies.app](https://bitbuddies.app)** and generate your own animated character in under 5 minutes. Upload a photo, watch the pipeline work its magic, and get a transparent animated sprite you can use anywhere. With the Chrome extension, your Bit Buddy can run across all your web pages, following you around the internet.

And if something breaks? Don't worry—Temporal's got your back. (And my phone will buzz.)
