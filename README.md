# Contribution [1]: [Notifications for long running hooks]

**Contribution Number:** [1]  
**Student:** [Shriya Upadhyay]  
**Issue:** [[GitHub issue link](https://github.com/lakekeeper/lakekeeper/issues/1101)]  
**Status:** [Phase I] [Complete]

---

## Why I Chose This Issue

[1-2 paragraphs explaining why this issue interests you, how it matches your skills/learning goals, what you hope to learn]

This issue interests me because it involves Rust, a language I'm hoping to become more familiar with. Adding an observability feature seems like a great place to get started. I also appreciated how specific the prompt was, so I have a head start on replicating the issue and a clearer idea of where in the codebase to start. I don't have much experience in the data infrastructure space, but I'm excited to expand my skill set. My background is in Python and TypeScript building agentic AI systems, so understanding the infrastructure layer that supports those systems is something I'm genuinely motivated to learn. Through this project, I hope to get comfortable contributing to a large-scale codebase, communicate my changes clearly, and use AI tools thoughtfully rather than excessively.

---

## Understanding the Issue

### Problem Description

Lakekeeper is a metadata catalog for Apache Iceberg tables. It stores pointers to files and schemas, but not actual data. When a client performs an operation such as creating a table, Lakekeeper writes the change to Postgres in a transaction, commits the transaction, runs any event listeners (like notifications), and returns a response to the client. However, because event listeners run in between committing the transaction and returning a response, slow event listeners tend to slow down responses without giving any indication of the cause.

### Expected Behavior

When an event listener takes longer than a specific threshold to run, Lakekeeper should log the slow listener and its runtime. It should also record the runtime in a Prometheus histogram so it can be used for metrics and further reporting.

### Current Behavior

Event listener runtime is not measured at all.

### Affected Components

The issue referenced this file: crates/iceberg-catalog/src/service/endpoint_hooks.rs:72-340, which no longer exists. Now, the file is called crates/lakekeeper/src/service/events/dispatch.rs, and the type we are working with is EventDispatcher.

We will specifically be working with dispatch_event! which all event dispatchers utilize. Changing this macro will affect the timing measurements for all event listeners.

---

## Reproduction Process

### Environment Setup

1. Cloned my fork: `git clone https://github.com/shriya-upadhyay/lakekeeper.git`
2. Started Postgres via Docker:
```bash
   docker run -d --name postgres-16 -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:17
```
3. Created `.env` with connection strings and encryption key (see `.env.example` in repo)
4. Installed `sqlx-cli` and `cargo-sort`:
```bash
   cargo install sqlx-cli cargo-sort
```
5. **Challenge:** `cargo install cargo-nextest` failed due to a compile error. Fix: install the prebuilt binary instead:
```bash
   curl -LsSf https://get.nexte.st/latest/mac | tar zxf - -C ${CARGO_HOME:-~/.cargo}/bin
```
6. **Challenge:** First test run failed with `is cmake not installed?` — `protobuf-src` compiles a C++ library from source and needs CMake:
```bash
   brew install cmake
```
7. Installed `just` task runner (needed for `just check-clippy`, `just fix-format`):
```bash
   brew install just
```
8. Ran migrations and tests:
```bash
   sqlx database create
   sqlx migrate run --source crates/lakekeeper-storage-postgres/migrations
   cargo nextest run --all-features  
   just check-clippy                  
```


### Steps to Reproduce

Since this isn't more a bug, but rather a missing observability feature, I just looke dthrough the files to see where the missing code could be implemented:

1. Open `crates/lakekeeper/src/service/events/dispatch.rs`
2. Locate the `dispatch_event!` macro — this is where all event listeners are invoked
3. Note that listener calls are not wrapped in any timing logic

### Reproduction Evidence

- Branch: [fix-issue-1101](https://github.com/shriya-upadhyay/lakekeeper/tree/fix-issue-1101)

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

The lack of timing code wrapping the dispatch event. 

### Proposed Solution

Will add timing around each event listener call in dispatch event to keep track of time before each event and how much time has elapsed afterward. Will use a histogram to record timing results.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Event listeners run after transaction commit but before response. Slow listeners silently increase client-visible latency. Currently, there is no measurement of listener runtime in logs or metrics.

**Match:** Existing patterns to follow:
- `cache_metrics.rs` — shows how to declare and describe a histogram with `LazyLock` + `describe_histogram!`
- `role_assignment.rs` — shows the `metrics::histogram!(NAME, "label" => value).record(x)` call pattern and `warn!` on threshold
- The `dispatch_event!` macro itself is the insertion point; no new types need to be created

**Plan:** [Step-by-step implementation plan]
1. In `dispatch_event!` in `dispatch.rs`: capture `Instant::now()` before each `listener.$method(...)` call, compute `.elapsed()` after `await`
2. Record elapsed seconds into a histogram: `metrics::histogram!("lakekeeper_event_listener_duration_seconds", "event" => stringify!($method)).record(elapsed.as_secs_f64())`
3. Add `if elapsed > SLOW_LISTENER_THRESHOLD { warn!(...) }` — log listener type name and duration
4. Add `describe_histogram!` registration in the appropriate metrics init location
5. Add a unit test: mock `EventListener` that sleeps, confirm histogram records and warn fires

**Implement:** [Link to branch — in progress](https://github.com/shriya-upadhyay/lakekeeper/tree/fix-issue-1101)

**Review:** 
Before submitting PR:
- [ ] `cargo nextest run --all-features` passes
- [ ] `just check-clippy` clean
- [ ] `just fix-format` applied
- [ ] PR title follows Conventional Commits: `feat(events): add timing and prometheus metrics to event dispatch`
- [ ] CLA signed on GitHub
- [ ] PR description references `Closes #1101`

**Evaluate:** 

- Unit test: slow mock listener → `warn!` fires and histogram records non-zero value
- Unit test: fast listener → no `warn!`, histogram still records (near-zero) value
- `cargo nextest run` stays green
- `just check-clippy` stays clean

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
