# Claude Prompting Guide — Advanced Developer Edition

---

## Table of Contents

1. [Advanced Prompting Techniques](#1-advanced-prompting-techniques)
   - [Constraint-First Prompting](#constraint-first-prompting) · [Role + Adversarial Framing](#role--adversarial-framing) · [Stepwise Locking](#stepwise-locking-plan--confirm--execute) · [Negative Space](#negative-space-prompting) · [Delta Prompting](#delta-prompting) · [Comparative Generation](#comparative-generation)
2. [Debugging](#2-debugging)
   - [Full Error Pasting](#paste-the-full-error-not-a-summary) · [Intermittent Failures](#intermittent-failures--give-frequency-and-conditions) · [Silent Failures](#silent-failures-no-error-wrong-output) · [Regressions](#regression--tell-claude-what-changed) · [Concurrency Bugs](#concurrency-bugs--provide-threadprocess-model) · [Binary Search](#the-binary-search-prompt)
3. [Performance Optimization](#3-performance-optimization)
   - [Measurements First](#lead-with-measurements-not-feelings) · [Profiler Output](#profiler-output-first-then-code) · [N+1 Queries](#n1-queries--show-the-orm-model) · [Caching Strategy](#caching-strategy--ask-for-options--trade-offs) · [Benchmarking](#asking-for-a-benchmark-first)
4. [Code Review](#4-code-review)
   - [Scoping](#scope-the-review-or-you-get-generic-feedback) · [Adversarial](#adversarial-review--assume-a-malicious-caller) · [Severity Tiers](#asking-for-severity-tiers) · [Regression Focus](#diff-review--regression-focus) · [Test Review](#reviewing-tests-not-just-code)
5. [Refactoring](#5-refactoring)
   - [Contract Locking](#lock-the-contract-first) · [Extract Function](#extract-function--give-the-target-size) · [Pattern Migration](#design-pattern-migration--explain-why) · [Step-by-Step](#step-by-step-with-verification-points) · [Behavioral Equivalence](#validate-behavioral-equivalence-after)
6. [Writing Tests](#6-writing-tests)
   - [Bug History](#give-claude-the-bug-history-not-just-the-code) · [Testing Strategy](#specify-the-testing-strategy-upfront) · [Contract Testing](#test-the-contract-not-the-implementation) · [Edge Cases](#boundary-and-edge-case-generation) · [TDD](#tdd--tests-before-code) · [Property-Based](#property-based-tests)
7. [API Design](#7-api-design)
   - [Consumer-First](#start-with-consumers-not-resources) · [Trade-offs](#enumerate-trade-offs-before-committing) · [Error Responses](#error-response-design) · [Versioning](#versioning-strategy) · [Breaking Change Audit](#breaking-change-audit)
8. [System Design](#8-system-design)
   - [Real Numbers](#anchor-every-design-to-real-numbers) · [Trade-off Tables](#trade-off-table-before-architecture) · [Failure Modes](#failure-mode-analysis-first) · [Database Access Patterns](#database-design--give-access-patterns) · [Zero-Downtime Migration](#migration-design--zero-downtime)
9. [Architecture Decision Records](#9-architecture-decision-records)
   - [Draft From Conversation](#draft-an-adr-from-a-conversation) · [Stress-Test a Draft](#stress-test-a-draft-adr) · [Rejected Options](#adr-for-a-rejected-option) · [Superseding](#superseding-an-old-adr) · [Completeness Review](#adr-review-for-completeness)
10. [Documentation Writing](#10-documentation-writing)
    - [Reader-First](#specify-the-reader-not-just-the-topic) · [README](#readme--give-claude-the-projects-job) · [Runbooks](#runbook--structured-for-3am-incidents) · [Stale Docs](#update-stale-documentation) · [Migration Guides](#migration-guide)
11. [Git Workflows](#11-git-workflows)
    - [Commit Messages](#commit-message-writing) · [Branch Strategy](#branch-strategy-design) · [Bisect](#git-bisect--finding-a-regression) · [Cleaning Up Branches](#cleaning-up-a-messy-branch) · [PR Descriptions](#pr-description-writing)
12. [Security Reviews](#12-security-reviews)
    - [Threat Modeling](#threat-model-before-code) · [OWASP Checklist](#owasp-top-10-checklist-review) · [Secrets Audit](#secrets-and-credential-audit) · [Auth Flow](#auth-flow-review) · [Input Validation](#input-validation-review)
13. [Incident Post-Mortems](#13-incident-post-mortems)
    - [Timeline](#build-the-timeline-first) · [Root Cause Analysis](#root-cause-analysis) · [Action Items](#action-items--specific-and-assignable) · [Write-Up](#post-mortem-write-up) · [Pattern Detection](#detecting-patterns-across-incidents)
14. [Explaining Code](#14-explaining-code)
    - [Level-Specific](#explain-to-a-specific-level) · [Mental Models](#build-a-mental-model-not-a-line-by-line-walk) · [Design Patterns](#explain-a-design-pattern-in-context) · [Why Not What](#explain-why-not-what) · [Onboarding](#explain-a-codebase-for-onboarding)
15. [Database Migrations](#15-database-migrations)
    - [Zero Downtime](#design-a-zero-downtime-migration) · [Column Rename](#rename-a-column-without-downtime) · [Backfill Strategy](#backfill-strategy-for-large-tables) · [Index Migration](#index-migration) · [Rollback Design](#rollback-design)
16. [Dependency Upgrades](#16-dependency-upgrades)
    - [Breaking Change Analysis](#breaking-change-analysis-before-upgrading) · [Upgrade Impact](#upgrade-impact-on-our-codebase) · [Security Patches](#security-patch--fast-path) · [Evaluating New Deps](#evaluating-a-new-dependency)

---

**[Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)**
— Bad→Good table · Prompt templates · Top 10 habits · Anti-patterns · Built-in commands

---

## 1. Advanced Prompting Techniques

### Constraint-First Prompting
Lead with what must NOT change. Claude defaults to "helpful" — which means it refactors, renames, adds error handling, and cleans up unless you lock it down.

```
❌ "optimize this database query"

✅ "optimize this query for read latency.
    Constraints:
    - Do not change the return shape — callers depend on column order
    - No new indexes (production freeze until Friday)
    - Must stay compatible with PostgreSQL 13
    
    SELECT u.id, u.email, p.plan_name
    FROM users u JOIN plans p ON u.plan_id = p.id
    WHERE u.created_at > NOW() - INTERVAL '30 days'"
```

### Role + Adversarial Framing
Ask Claude to argue against itself. Forces it to surface hidden failure modes before you're in production.

```
"You are a senior backend engineer doing code review.
 Find every way this auth middleware could fail or be bypassed.
 Be adversarial — assume the caller is malicious.
 
 [paste code]"
```

Follow up:
```
"Now fix the top 3 issues you found. Minimal changes only —
 don't restructure the function."
```

### Stepwise Locking (Plan → Confirm → Execute)
For non-trivial changes, force a planning step before any code is written.

```
"I want to migrate our user auth from JWT to session cookies.
 
 Step 1 only: List every file that will need to change and why.
 Do NOT write any code yet. I'll confirm the list before you proceed."
```

After review:
```
"Good. Now execute step by step, one file at a time.
 Pause after each file and wait for me to say 'next'."
```

### Negative Space Prompting
Describe what the output should NOT be.

```
"Refactor this service to use the repository pattern.

 Do NOT:
 - Add an interface/abstract class unless there are multiple implementations
 - Use dependency injection — we don't have a DI container
 - Split into more than 2 files
 - Change any method signatures that are called from tests
 
 [paste code]"
```

### Evidence-First Debugging
Give Claude a falsifiable hypothesis to evaluate.

```
"This function intermittently returns None. My hypothesis is that
 the race condition is in the cache write at line 47, not the fetch.
 
 Either confirm this hypothesis with evidence from the code,
 or tell me specifically why I'm wrong and what the actual cause is.
 
 [paste code + stack trace]"
```

### Delta Prompting
When iterating, describe only what changed from the last working state.

```
"The previous solution worked except for one case:
 when user.role is None (not 'guest', actually None),
 the permission check throws AttributeError instead of denying.
 
 Fix only that case. The rest of the logic is correct."
```

### Persona Switching for Different Lenses
Same code, different reviewers, different insights.

```
"Review this API design as a:
 1. Security engineer — what can an attacker exploit?
 2. Mobile developer consuming this API — what's painful?
 3. Junior dev who'll maintain this in 6 months — what's confusing?
 
 Give each perspective separately."
```

### Format Anchoring
Claude mirrors your format. Structure your input to get structured output.

```
"For each function below, produce output in exactly this format:

Function: <name>
Risk: HIGH | MEDIUM | LOW
Reason: <one sentence>
Fix: <code snippet or 'none needed'>

---
[paste functions]"
```

### Context Priming for Long Sessions
Front-load all invariants at the start of a complex session.

```
"Before we start: here are the constraints that apply to ALL
 code in this session unless I explicitly override:
 
 - Language: Python 3.11
 - No new pip dependencies
 - All functions must have type annotations
 - Error handling: raise exceptions, never return None on failure
 - Tests live in tests/ and use pytest
 
 Confirm you've noted these, then I'll give you the first task."
```

### Comparative Generation
Ask for multiple implementations and a verdict.

```
"Implement the rate limiter two ways:
 A) In-process using a token bucket
 B) Redis-backed using a sliding window
 
 After both implementations, give me a decision matrix:
 - Concurrency safety
 - Ops complexity  
 - Accuracy under burst load
 - Cost at 10k rps
 
 Then recommend one with a one-sentence justification."
```

### Minimal Reproduction Requests
```
"Write the smallest possible pytest test that would have caught
 the bug I just fixed in auth/token.py line 83.
 
 Requirements:
 - No mocks — use real objects
 - Must fail on the old code and pass on the new code
 - Under 20 lines"
```

### Chain-of-Custody for Diffs
```
"Refactor this module. After every logical change, add a comment
 in the format: # CHANGED: <what and why>
 
 I'll strip those comments after review — they're just for audit."
```

### The Correction Loop
When Claude gives a wrong answer, point at the specific error and why it's wrong.

```
❌ "that's not right, try again"

✅ "incorrect — you assumed the mutex is held before entering
    the function (line 3 of your solution), but in our code
    it's acquired inside. Fix only that assumption."
```

---

## 2. Debugging

### Paste the Full Error, Not a Summary
```
❌ "I'm getting a key error somewhere in the user lookup"

✅ "Getting this in production:

    KeyError: 'user_id'
    Traceback (most recent call last):
      File "api/routes/auth.py", line 47, in verify_token
        user_id = payload['user_id']
    
    JWT payload is: {'sub': '123', 'exp': 1715000000, 'iat': 1714996400}
    
    Token was issued by our old auth service before the migration.
    What's failing and how do I handle both payload shapes?"
```

### Intermittent Failures — Give Frequency and Conditions
```
"This fails roughly 1 in 50 requests, always under load, never in dev.

    AssertionError: expected 1 row, got 0
    File "db/user_repo.py", line 31, in get_by_email

 The function:
 [paste code]

 My hypothesis: it's a read-after-write race on the replica.
 Confirm or disprove. If I'm wrong, tell me what in the code
 points to a different cause."
```

### Silent Failures (No Error, Wrong Output)
```
"No exception is raised, but the output is wrong.

 Input:   {'amount': 99.99, 'currency': 'GBP', 'user_id': 42}
 Expected: charge in GBP, converted to USD at current rate
 Actual:   charge in USD, no conversion applied

 Trace through this function step by step and identify
 where it stops doing the right thing:

 [paste function]"
```

### Regression — Tell Claude What Changed
```
"This worked in v1.3.2, broke in v1.4.0. Here's the git diff:

 [paste diff]

 The failing test:

 [paste test + output]

 Don't look beyond the diff — the bug is in those changes.
 Which line introduced the regression and why?"
```

### Performance Debugging — Be Specific About Measurements
```
"This endpoint went from p95=120ms to p95=800ms after last deploy.
 Profiler output:

   97% of time in db/queries.py:get_user_with_permissions (line 88)
   Before: ~2 queries. After: seeing 40-60 queries per request.

 This looks like N+1 but I don't see a loop. What am I missing?

 [paste get_user_with_permissions + any ORM model definitions it touches]"
```

### Memory / Resource Leaks
```
"Process memory grows ~50MB/hour and never releases.
 No obvious leaks — no globals being appended to.

 These are the only long-lived objects in the app:
 [paste relevant classes/singletons]

 What should I instrument first to find the leak?
 Give me 3 specific things to add — logging, tracemalloc,
 or object counts — in order of likelihood."
```

### Concurrency Bugs — Provide Thread/Process Model
```
"Race condition, hard to reproduce. Setup:
 - 4 gunicorn workers, no threading (gevent)
 - Shared Redis cache
 - Bug: two workers occasionally write the same invoice_id

 The ID generation logic:
 [paste code]

 Is this actually a race? If yes, show me the exact interleaving
 that produces a collision. If no, tell me what else could cause it."
```

### Asking for a Minimal Repro
```
"Write me a self-contained script (no app imports) that
 reproduces this deadlock.

 Context:
 - SQLAlchemy 2.0, PostgreSQL
 - Two threads, each doing: read row → compute → write row
 - Deadlock happens when they touch overlapping rows

 The repro should:
 - Use sqlite so it runs anywhere
 - Deadlock within 5 seconds
 - Print which thread is blocked"
```

### Reading a Stack Trace You Don't Own
```
"I didn't write this stack trace — it's inside SQLAlchemy internals.
 Help me find which line in MY code triggered it.

 [paste full 40-line stack trace]

 My code starts at: myapp/db/session.py line 23
 What did I do wrong and what's the fix?"
```

### The Binary Search Prompt
```
"I know the bug is somewhere in this 200-line function.
 Tell me: if you had to split it in half to bisect the bug,
 where would you cut it and what invariant would you assert
 at that midpoint to determine which half contains the bug?"
```

### Correcting a Wrong Fix
```
"Your fix didn't work. Here's what happened:

 Before your change: KeyError on line 47
 After your change:  TypeError: 'NoneType' object is not subscriptable
                     same line 47

 You handled the missing key but didn't handle the case where
 the key exists and its value is None.
 Fix only that. The rest of your change was correct."
```

### Asking for Debug Instrumentation, Not a Fix
```
"Don't fix this yet. Add logging statements so I can see:
 1. The value of `session` at line 12 before the check
 2. Which branch of the if/else at line 34 is taken
 3. The return value before it exits

 Use Python's logging module, level=DEBUG.
 I'll run it and share the output before we discuss a fix."
```

### The Debugging Meta-Pattern
Structure every debug prompt as:
```
1. What you observed       (exact error or output)
2. When it happens         (always / intermittent / under load)
3. What changed recently   (deploy, config, data volume)
4. Your current hypothesis (even if wrong)
5. What you want           (diagnosis / fix / instrumentation)
```

---

## 3. Performance Optimization

### Lead with Measurements, Not Feelings
```
❌ "this query is slow, can you optimize it"

✅ "Query runtime: p50=2.1s, p99=8.4s on a table with 12M rows.
    EXPLAIN ANALYZE output:

    Seq Scan on events (cost=0.00..340821.00 rows=12000000)
                        actual time=0.1..3821.4 rows=12000000
    Filter: (user_id = $1 AND created_at > '2026-01-01')
    Rows removed by filter: 11,987,432

    [paste query]

    Target: p99 under 200ms. No schema changes — migration freeze.
    What index would fix this and what's the trade-off on write throughput?"
```

### Profiler Output First, Then Code
```
"py-spy output (sampled at 100hz, 60s window):

    42% db/repo.py:get_permissions     line 88
    31% db/repo.py:get_permissions     line 91  
    18% auth/middleware.py:verify      line 34
     9% everything else

 Both hot lines are inside a loop. Here's the function:

 [paste code]

 Fix the hot path only. Don't touch verify() — that's owned
 by another team."
```

### N+1 Queries — Show the ORM Model
```
"Seeing 60-80 queries per request. Should be 2-3.
 Django Debug Toolbar shows them all as variations of:

    SELECT * FROM permissions WHERE role_id = %s

 Called once per user in the response list (~40 users).

 Models involved:
 [paste User, Role, Permission models]

 View code:
 [paste view]

 Fix the N+1 with select_related or prefetch_related.
 Show me the corrected queryset and explain why you chose one over the other."
```

### Caching Strategy — Ask for Options + Trade-offs
```
"This function is called 800 times/second, result changes once per hour.
 Runtime: 340ms (hits Postgres + Redis + an external API).

 [paste function signature and body]

 Give me three caching strategies ranked by implementation complexity:
 1. Simplest possible
 2. Correct under concurrent stampede
 3. Correct + observable (cache hit/miss metrics)

 For each: show the code change, state the cache key, and tell me
 what breaks if the cache gets stale."
```

### Algorithmic Complexity — Give Input Scale
```
"This runs fine at 1k items, times out at 100k.

 [paste function]

 What's the current time complexity? What's the bottleneck?
 Rewrite it to O(n log n) or better. If that's not possible
 without changing the output contract, tell me what the limit is."
```

### Memory Optimization — Give the Heap Profile
```
"tracemalloc output after processing 10k records:

    Top allocations:
    api/transform.py:88: size=2.1 GiB, count=9,847,200, average=224 B
    api/transform.py:91: size=890 MiB, count=3,291,000, average=284 B

 The transform function:
 [paste code]

 It processes a CSV row-by-row. Why is it holding 2GB?
 Rewrite it to stay under 50MB regardless of file size."
```

### Concurrency / Parallelism
```
"This pipeline processes 500 items sequentially, takes 45 seconds.
 Each item makes 3 independent HTTP calls (~90ms each).
 No shared state between items.

 [paste pipeline code]

 Rewrite using asyncio to process items concurrently.
 Constraints:
 - Max 20 concurrent requests (API rate limit)
 - Must preserve output order
 - Failures on individual items should not abort the batch"
```

### Asking for a Benchmark First
```
"Before we optimize, write a benchmark for this function.

 [paste function]

 Requirements:
 - Use pytest-benchmark
 - Test with realistic input sizes: 100, 1k, 10k, 100k items
 - Parameterized so I can see the scaling curve
 - Warm up the cache before measuring

 I'll run this before and after any change to validate improvement."
```

### Database Schema — Give Cardinality
```
"Slow query:
 SELECT * FROM orders 
 WHERE status = 'pending' AND merchant_id = $1
 ORDER BY created_at DESC LIMIT 20;

 Table stats:
 - 50M rows total
 - status distribution: 97% 'completed', 2% 'pending', 1% other
 - ~500 distinct merchant_ids
 - created_at spans 3 years

 Current index: (merchant_id)
 
 What composite index should I add and in what column order?
 Show me the EXPLAIN plan you'd expect after the index exists."
```

### Front-End / Bundle Performance
```
"webpack-bundle-analyzer output shows:

    moment.js        67 KB (gzipped)   — used for 1 date format call
    lodash           71 KB (gzipped)   — used for _.groupBy only
    recharts        142 KB (gzipped)   — used on 1 route

 Entry point: src/App.tsx
 The 3 usage sites:
 [paste the 3 import lines + usage]

 For each:
 1. Replace moment/lodash with native alternatives if possible
 2. Lazy-load recharts since it's route-gated
 
 Show the exact import changes."
```

### Validate the Optimization Didn't Break Correctness
```
"You optimized the sort from O(n²) to O(n log n) using a heap.
 Before I ship this: write 5 test cases that would catch any
 behavioral difference between the old and new implementation,
 especially around:
 - Duplicate values
 - Empty input
 - Single element
 - Already-sorted input
 - Stability (equal elements preserve original order)"
```

### The Optimization Conversation Template
```
Current behavior:   [metric — latency, throughput, memory, CPU%]
Target:             [specific number, not "faster"]
Profiler evidence:  [where time/memory actually goes]
Constraints:        [no schema changes / no new deps / must be thread-safe]
Code:               [paste the hot path only, not the whole file]
Question:           [diagnose / rewrite / suggest strategy]
```

---

## 4. Code Review

### Scope the Review or You Get Generic Feedback
```
❌ "review this code"

✅ "Review this for security issues only — specifically:
    - Input validation at the API boundary
    - SQL injection surface
    - Auth bypass possibilities
    
    Ignore style, naming, and performance unless they create a security risk.
    
    [paste code]"
```

### Adversarial Review — Assume a Malicious Caller
```
"Review this API endpoint as if you're trying to abuse it.
 Assume the caller controls all inputs including headers.
 
 For each vulnerability: show the exact malicious payload,
 the impact, and the minimal fix.
 
 [paste endpoint handler]"
```

### Pre-PR Checklist Review
```
"Review against our team checklist before I open the PR:

 Must pass:
 □ No raw SQL — use the query builder
 □ All new endpoints have rate limiting via @rate_limit decorator  
 □ Database calls not in a loop
 □ No secrets or PII logged
 □ New behavior covered by at least one integration test

 Flag any checkbox that fails with the specific line.
 Skip commentary on things that pass.
 
 [paste diff]"
```

### Diff Review — Regression Focus
```
"Review this diff for regressions only.
 The old code worked. Tell me what the new code breaks.

 Pay attention to:
 - Changed return values or exceptions callers might depend on
 - Removed null checks
 - Altered control flow in error paths
 - Any behavior difference under concurrent access

 [paste git diff]"
```

### Architecture Review — Give the Constraints
```
"Review this service design for correctness under our constraints:

 - Deployed as 8 stateless pods behind a load balancer
 - No sticky sessions
 - Redis available but Postgres is the source of truth
 - SLA: 99.9% uptime, max 500ms p99

 Specifically: does this design have any failure modes that
 violate the SLA? What happens when one pod dies mid-request?

 [paste architecture / key classes / sequence diagram]"
```

### API Design Review — From the Consumer's Perspective
```
"Review this REST API design as a mobile developer who will consume it.

 Flag anything that will cause pain:
 - Chatty endpoints requiring multiple calls for one screen
 - Missing fields that will require a second request
 - Pagination design that breaks on new inserts
 - Error shapes that are hard to display to users
 - Breaking change risk if we add fields later

 [paste OpenAPI spec or route definitions]"
```

### Asking for Severity Tiers
```
"Review this code and bucket your findings into:

 BLOCKING   — must fix before merge (correctness, security)
 IMPORTANT  — should fix this sprint (reliability, maintainability)  
 MINOR      — fix when touching this file again
 NITPICK    — skip if you disagree

 Be conservative with BLOCKING. If you're unsure, downgrade it."
```

### Reviewing Tests, Not Just Code
```
"Review the test suite for this module — not the implementation.

 Flag:
 - Tests that couldn't possibly catch the bug they claim to test
   (wrong assertion, mocked too deep)
 - Missing cases: what input would break the code that no test covers?
 - Tests coupled to implementation details that will break on refactor
 - Any test that passes on buggy code (write a counterexample if possible)

 [paste tests]"
```

### Incremental Review on a Large PR
```
"This PR is 800 lines. Review in this order of priority:

 1. auth/middleware.py — highest risk, review thoroughly
 2. api/routes/*.py — check for input validation only
 3. db/migrations/ — check for data loss or lock risk
 4. Everything else — skim for obvious issues only

 Stop and flag immediately if you find anything BLOCKING in (1)."
```

### Before/After Review — Validate a Refactor
```
"I refactored this function. Confirm the behavior is identical.

 BEFORE:
 [paste original]

 AFTER:
 [paste refactor]

 Specifically check:
 - Same return type and shape for all inputs
 - Same exceptions raised in same conditions
 - No change in side effects (logging, metrics, DB writes)
 
 If behavior differs anywhere, show the input that produces
 different output."
```

### Asking for the Single Most Important Finding
```
"You have 2 minutes to review this PR. What's the one thing
 most likely to cause an incident in production?
 
 One finding only. Be specific about why it would fail and when.
 
 [paste code]"
```

### Getting Claude to Write the Review Comment
```
"Format your findings as GitHub PR review comments.
 For each issue use this format:

 **File:** path/to/file.py  
 **Line:** 47  
 **Severity:** BLOCKING | IMPORTANT | MINOR  
 **Comment:** [the comment text, as if written to a colleague]

 Be direct — write the comment I'd paste into GitHub, not
 a meta-description of what the comment should say."
```

### Self-Review Before Asking Claude
```
"I've already reviewed this. My concerns are:
 1. The lock on line 34 might not cover the full critical section
 2. Not sure if the retry logic handles idempotency correctly

 Validate or disprove each concern with evidence from the code.
 Then flag anything I missed that's worse than what I found.

 [paste code]"
```

### The Review Prompt Template
```
Context:     [what this code does, who calls it]
Risk areas:  [auth / data integrity / concurrency / external calls]
Review for:  [security / correctness / regression / API design]
Skip:        [style / naming / things you don't own]
Format:      [severity tiers / GitHub comments / bullet list]
Code:        [paste]
```

### Built-in Claude Code Commands
```bash
/review           # reviews the current branch diff
/security-review  # focused security pass on changed files  
/ultrareview      # multi-agent deep review (use before merging)
```

---

## 5. Refactoring

### Lock the Contract First
```
❌ "refactor this class to be cleaner"

✅ "Refactor this class. The following must not change:
    - Public method signatures (called from 14 other files)
    - Exception types raised (callers catch specific exceptions)
    - Return value shapes
    - Side effects: logging calls and the metrics.increment() calls
    
    You can change anything internal.
    
    [paste class]"
```

### Extract Function — Give the Target Size
```
"This function is 180 lines. Break it into smaller functions.

 Rules:
 - Each extracted function must be testable in isolation
 - No function longer than 30 lines after the refactor
 - Private helpers prefixed with underscore
 - Don't extract things that are only called once unless
   the name makes the code significantly clearer
 
 [paste function]"
```

### Remove Duplication — Show All the Copies
```
"These three functions are 80% identical. Consolidate them.

 Differences between the three:
 - build_user_query filters by user_id
 - build_admin_query filters by org_id and adds a JOIN
 - build_report_query has no filter but adds GROUP BY

 Don't use **kwargs to paper over the differences — make the
 variation explicit in the unified function's signature.

 [paste all three functions]"
```

### Design Pattern Migration — Explain Why
```
"Migrate this from procedural to the strategy pattern.

 Problem I'm solving: we're adding a 4th pricing engine next sprint
 and the current if/elif chain is untestable in isolation.

 Requirements:
 - Each strategy must be swappable at runtime (loaded from config)
 - Strategies must be independently unit-testable
 - Don't introduce an abstract base class — use duck typing
 
 [paste current pricing code]"
```

### Complexity Reduction — Give the Metric
```
"This function has a cyclomatic complexity of 23 (measured by radon).
 Target: under 10.

 Approach I want: replace the nested conditionals with early returns
 and a lookup table where possible. Do not extract helper functions —
 I want it readable in one place.

 [paste function]"
```

### Step-by-Step with Verification Points
```
"Refactor this module in 3 stages. After each stage, stop
 and show me only what changed so I can verify before continuing.

 Stage 1: Extract the validation logic into a separate function.
           No behavior change — pure structural move.
 Stage 2: Replace the manual validation with pydantic.
 Stage 3: Add the missing error cases we discussed.

 Start with Stage 1 only."
```

### Async Migration
```
"Convert this synchronous function to async.

 Context:
 - Python 3.11, using asyncio
 - httpx is already a dependency (use AsyncClient)
 - Callers are already async — just need this function to match
 - The retry logic must be preserved exactly

 Do NOT convert the database calls — those use a sync ORM
 that we're not migrating yet. Leave those as
 asyncio.to_thread() calls.

 [paste function]"
```

### Type Annotation Refactor
```
"Add type annotations to this module.

 Rules:
 - Use Python 3.10+ union syntax (X | Y, not Union[X, Y])
 - No 'Any' unless genuinely unavoidable — explain each use
 - Annotate public functions fully, private helpers minimally
 - If a return type is complex, define a TypedDict or dataclass
   rather than annotating as dict[str, Any]

 [paste module]"
```

### God Class Decomposition
```
"This class has 34 methods and 800 lines. Decompose it.

 I've identified these responsibility clusters:
 1. User authentication (methods: login, logout, verify_token)
 2. Permission checks (methods: can_read, can_write, get_role)
 3. Session management (methods: create_session, expire_session)

 Split into 3 classes. Requirements:
 - Existing callers use UserService.method() — preserve that
   by keeping a thin UserService facade that delegates
 - No circular imports
 - Show the import graph after the split

 [paste class]"
```

### Error Handling Refactor
```
"The error handling in this module is inconsistent:
 - Some functions return None on failure
 - Some raise ValueError
 - Some raise custom exceptions
 - One swallows exceptions silently

 Standardize to: raise specific custom exceptions, never return
 None for error cases, never swallow.

 Define the exception hierarchy first, then show each changed
 function. Callers are in api/routes/ — I'll update those separately."
```

### Configuration / Magic Number Cleanup
```
"Extract all magic numbers and hardcoded strings into named constants.

 Rules:
 - Group related constants into a class or module, not scattered globals
 - Name constants for what they mean, not what they are
   ('MAX_LOGIN_ATTEMPTS' not 'THREE')
 - If a value appears in multiple files, it goes in config/constants.py
 - Don't extract things that are obviously local (loop indices, etc.)

 [paste files]"
```

### Validate Behavioral Equivalence After
```
"You just refactored the payment processor. Before I run the
 test suite, identify the 3 input cases most likely to expose
 a behavioral difference between the old and new code.

 For each: show the input, the expected output from both versions,
 and why this case is high-risk."
```

### The Refactor Prompt Template
```
Goal:        [what problem the refactor solves — not just "cleaner"]
Constraints: [public API / exceptions / side effects that must not change]
Approach:    [pattern / technique you want applied]
Out of scope:[explicit list of what to leave alone]
Validation:  [how you'll verify no behavior change]
Code:        [paste]
```

---

## 6. Writing Tests

### Give Claude the Bug History, Not Just the Code
```
❌ "write tests for this function"

✅ "Write tests for this function. Known bugs from production:
    - Jan 2025: returned wrong total when discount > item price
    - Mar 2025: crashed on empty cart (None vs empty list confusion)
    - Last week: tax calculation wrong for orders spanning midnight UTC

    Cover those cases explicitly, plus the happy path.
    Use pytest. No mocks — use real objects.

    [paste function]"
```

### Specify the Testing Strategy Upfront
```
"Write tests for this payment processor.

 Strategy:
 - Unit tests for pure calculation logic (no I/O)
 - Integration tests for the Stripe calls using responses library
   to record/replay — no live API calls, no mocking internals
 - One end-to-end test that hits our test Stripe account
   (mark with @pytest.mark.e2e so CI can skip it)

 Do NOT mock the database — use a test Postgres instance.
 Fixture for that is already in conftest.py as 'db_session'.

 [paste payment processor]"
```

### Test the Contract, Not the Implementation
```
"Write tests that validate the public contract of this function,
 not its implementation.

 The contract:
 - Given valid input, returns a dict with keys: id, status, created_at
 - Given invalid email, raises ValueError with message containing 'email'
 - Given duplicate username, raises UsernameConflictError
 - Is idempotent: calling twice with same input returns same result

 If your tests would break when I refactor the internals without
 changing the contract, rewrite them until they wouldn't.

 [paste function]"
```

### Boundary and Edge Case Generation
```
"Generate edge case tests for this input validation function.

 Think through:
 - Numeric boundaries (0, -1, MAX_INT, float('inf'), NaN)
 - String edge cases (empty, whitespace-only, Unicode, null bytes,
   very long strings, SQL injection strings, HTML tags)
 - Type coercion surprises (1 vs '1' vs True vs 1.0)
 - None vs missing vs explicitly-set-to-None

 Format as pytest.mark.parametrize. Include a comment on each
 case explaining what invariant it's testing.

 [paste validation function]"
```

### TDD — Tests Before Code
```
"Write the tests first. I'll implement the function after.

 Function to implement: parse_connection_string(s: str) -> ConnectionConfig

 Expected behavior:
 - Parses 'postgres://user:pass@host:5432/dbname'
 - Parses 'postgres://user@host/dbname' (no password, no port)
 - Raises ValueError on missing scheme
 - Raises ValueError on missing host
 - Returns a dataclass with fields: scheme, user, password, host, port, dbname
   (password and port are Optional)

 Write tests that will fail until the implementation is correct.
 Make them the specification — I should be able to implement from
 the tests alone without reading this message."
```

### Regression Test From a Bug Report
```
"Write a regression test for this bug before I fix it.

 Bug: when two users submit orders at the same millisecond,
 both get the same order_id.

 Requirements for the test:
 - Must FAIL on the current code
 - Must PASS after the fix
 - Must be deterministic — no sleep() or timing-dependent assertions
 - Use threading to force the race condition reliably

 [paste order ID generation code]"
```

### Property-Based Tests
```
"Write property-based tests using Hypothesis for this serializer.

 Properties that must hold for ALL inputs:
 1. serialize(deserialize(x)) == x  (round-trip)
 2. deserialize(serialize(x)) == x
 3. serialize() never raises — invalid input returns an error object
 4. Output is always valid UTF-8

 Use @given with appropriate strategies. For the User model,
 define a custom strategy that generates realistic but arbitrary users.

 [paste serializer + User model]"
```

### Mock Boundary — Explicit About What to Mock
```
"Write unit tests for this service class.

 Mock exactly these external calls and nothing else:
 - stripe.PaymentIntent.create() — return a fixture response
 - send_email() — assert it's called with correct args, don't send
 - datetime.now() — control time in tests that check expiry logic

 Do NOT mock:
 - The database (use the test db fixture from conftest.py)
 - Internal methods of the class under test
 - Any pure functions

 [paste service class]"
```

### Test Coverage Gap Analysis
```
"Here's a function and its existing tests.
 Tell me what's not covered — don't write the missing tests yet.

 Format:
 - Path not covered: [describe the code path]
 - Input that triggers it: [concrete example]
 - Risk if untested: [what bug could hide here]

 After I review the gaps, I'll tell you which ones to fill in.

 [paste function + existing tests]"
```

### Fixture Design
```
"Design the test fixtures for this test suite.

 The system under test has:
 - User (has a Role, has many Orders)
 - Order (has many LineItems, has one PaymentRecord)
 - PaymentRecord (has a status: pending/captured/refunded)

 I need fixtures for:
 - A user with no orders
 - A user with one pending order
 - A user with a mix of captured and refunded orders
 - An admin user

 Use pytest fixtures with appropriate scope (function vs session).
 Make fixtures composable — 'order_with_payment' should build on
 'basic_order', not duplicate it."
```

### Async Tests
```
"Write pytest-asyncio tests for this async service.

 Setup:
 - pytest-asyncio in auto mode (asyncio_mode = 'auto' in pytest.ini)
 - Use httpx.AsyncClient for HTTP calls, not aiohttp
 - Database: use aiosqlite with an in-memory DB per test

 Tests needed:
 1. Successful round-trip through the full async pipeline
 2. Timeout handling when the downstream service is slow
    (use anyio.move_on_after to simulate)
 3. Correct behavior when two coroutines race to write the same key

 [paste async service]"
```

### Parametrize Aggressively
```
"Rewrite these 12 near-identical test functions as a single
 parametrized test.

 [paste 12 tests]

 Rules:
 - Each test case needs an ID string so failures are readable
 - Group cases by what they're testing, not arbitrarily
 - If some cases need different setup, use indirect parametrize
   rather than if/else inside the test body"
```

### Validating Claude's Tests
```
"You just wrote 8 tests. For each one:
 1. Show me an obviously buggy implementation that would still pass it
 2. If one exists, fix the test to catch that bug

 I want tests that act as a specification, not tests that
 just confirm the current behavior."
```

### The Test Prompt Template
```
What to test:    [function/class/endpoint]
Known bugs:      [past failures to explicitly cover]
Strategy:        [unit / integration / e2e — and why]
Mock boundary:   [exactly what to mock, what to leave real]
Framework:       [pytest / jest / go test + any plugins]
Don't:           [patterns to avoid — mocking internals, etc.]
Validate:        [how you'll confirm tests catch real bugs]
```

### The One Rule
```
"After writing the tests, introduce this bug into my code:
 [describe a plausible bug]
 
 At least one of your tests must fail. If none do, the tests
 are not testing what they claim."
```

---

## 7. API Design

### Start With Consumers, Not Resources
```
❌ "design a REST API for our orders system"

✅ "Design a REST API for our orders system.

    Consumers and their primary needs:
    - Mobile app: needs order list + detail in one call (slow networks)
    - Admin dashboard: needs filtering, sorting, bulk operations
    - Warehouse system: polls for 'pending' orders every 30s
    - Webhooks: 3rd parties need to be notified on status change

    Constraints:
    - Public API — versioning required from day one
    - SLA: 200ms p99 for read endpoints
    - We're on Postgres, no search engine

    Give me the endpoint list first. No implementation yet."
```

### Enumerate Trade-offs Before Committing
```
"I need to design pagination for a feed endpoint.
 Give me three approaches:
 1. Offset-based
 2. Cursor-based
 3. Keyset

 For each: show the request/response shape, then a table:
 - Consistent under concurrent inserts?
 - Works with arbitrary sort order?
 - Can jump to page N?
 - Index-friendly?
 - Complexity to implement?

 Recommend one given: feed is append-heavy, clients need
 'load more', jumping to page N is not needed."
```

### Error Response Design
```
"Design the error response schema for our API.

 Requirements:
 - Machine-readable: clients must be able to handle errors
   programmatically without parsing the message string
 - Human-readable: message is safe to show in a UI
 - Debuggable: enough info to trace the request in our logs
 - Consistent: same shape for validation errors, auth errors,
   server errors, rate limit errors

 Show me:
 1. The JSON schema
 2. One example for each error category
 3. Which HTTP status codes map to which error types
 4. How validation errors on multiple fields are represented"
```

### Versioning Strategy
```
"Design a versioning strategy for our public API.

 Context:
 - Currently at v1, about to ship breaking changes for v2
 - We must support v1 for 18 months after v2 ships
 - 3rd party developers have built on v1
 - Team size: 6 engineers, can't run parallel codebases indefinitely

 Evaluate:
 A) URL versioning (/v1/, /v2/)
 B) Header versioning (API-Version: 2024-01)
 C) Content negotiation (Accept: application/vnd.api+json;version=2)

 For each: maintenance burden, client experience, and how we
 sunset v1 without breaking integrations.

 Recommend one. Include the deprecation notice strategy."
```

### Authentication and Authorization Design
```
"Design auth for our API. Two consumer types:

 1. First-party mobile/web app (our own frontend)
 2. Third-party developers (building integrations)

 Requirements:
 - First-party: short-lived tokens, silent refresh, works offline
 - Third-party: long-lived API keys, scoped permissions,
   revocable per-key, audit log of usage
 - Both: rate limited per identity, not per IP

 Design the token/key format, the auth flow for each consumer type,
 and the permission model. Flag any security trade-offs explicitly."
```

### Breaking Change Audit
```
"Audit this proposed change for breaking changes.

 Current API response:
 [paste current JSON]

 Proposed new response:
 [paste new JSON]

 A change is breaking if a client written against v1 would:
 - Throw an exception
 - Silently produce wrong behavior
 - Need to change code to keep working

 Categorize each difference:
 - BREAKING: client must update
 - NON-BREAKING ADDITIVE: client can ignore safely
 - BEHAVIORAL: shape is same but meaning changed (most dangerous)

 For each BREAKING change, suggest a non-breaking alternative."
```

### The API Design Prompt Template
```
Consumers:    [who calls this and what they need]
Constraints:  [latency SLA / versioning / public vs internal]
Context:      [data store / team size / existing patterns]
Design for:   [new API / extending existing / breaking change review]
Deliver:      [endpoint list / full spec / trade-off analysis / review]
Don't:        [patterns or services off the table]
```

---

## 8. System Design

### Anchor Every Design to Real Numbers
```
❌ "design a scalable notification system"

✅ "Design a notification system.

    Scale:
    - 2M active users
    - Peak: 50k notifications/second (flash sales)
    - Steady state: 500/second
    - Channels: push, email, SMS, in-app
    - Delivery SLA: push < 5s, email < 30s, SMS < 60s

    Constraints:
    - Must not send duplicates (user complaints are high-severity)
    - Opt-out must propagate in < 1 minute
    - Budget: can't add more than 2 new managed services

    Start with the data flow diagram, then component breakdown."
```

### Trade-off Table Before Architecture
```
"I need to choose between event-driven and request-response
 for our order processing pipeline.

 Context:
 - 10 downstream services need to react to order state changes
 - Some need real-time (fraud detection), some can be async (email)
 - Team has deep HTTP knowledge, limited Kafka experience
 - Orders must never be lost, even if a downstream service is down

 Give me a trade-off table:
 Criteria | Event-driven | Request-response
 ---------|--------------|-----------------
 [fill in: consistency / latency / ops complexity /
  failure isolation / team ramp-up / debuggability]

 Then recommend one, with the one condition under which you'd switch."
```

### Failure Mode Analysis First
```
"Before designing this system, enumerate its failure modes.

 System: distributed rate limiter across 12 API pods using Redis.

 For each failure scenario, tell me:
 - What happens to traffic (blocked / passed / unpredictable)
 - Whether it fails open or closed
 - Detection time
 - Recovery path

 Scenarios:
 1. Redis goes down completely
 2. Redis latency spikes to 500ms
 3. Network partition between 4 pods and Redis
 4. One pod has a stale Redis connection it doesn't know about
 5. Clock skew > 1 second between pods

 After the analysis, propose a design that handles all five."
```

### Database Design — Give Access Patterns
```
"Design the database schema for a multi-tenant SaaS analytics system.

 Access patterns (in order of frequency):
 1. Get all events for tenant X in time range T1-T2, paginated (read-heavy)
 2. Count events by type for tenant X, last 30 days (dashboard)
 3. Write single events at ~10k/s across all tenants
 4. Backfill historical events (bulk insert, ~1M rows at once)
 5. Delete all data for a churned tenant (GDPR)

 Constraints:
 - Postgres (already in stack, can't add ClickHouse yet)
 - Tenant data must be isolated (row-level security or schema-per-tenant)
 - Pattern (1) must return in < 100ms for up to 10M events per tenant

 Show the schema, indexes, and partitioning strategy."
```

### Migration Design — Zero Downtime
```
"Design a zero-downtime migration from our monolith to services.

 Monolith: Rails app, Postgres, 200k lines, 5 years old.
 Target: extract the billing domain first (highest change rate).

 Constraints:
 - Can't take downtime — 24/7 service, 3 regions
 - The billing DB tables are joined by 40+ other queries in the monolith
 - Team of 8, can't freeze feature work during migration
 - Must be able to roll back at each step

 Design the strangler fig sequence:
 - What's the order of steps?
 - Where's the dual-write phase and how long does it last?
 - How do we validate the new service's data matches the old?
 - What's the rollback trigger and rollback procedure?"
```

### The System Design Prompt Template
```
What:         [system to design / review / scale]
Numbers:      [current scale, target scale, SLAs]
Constraints:  [tech stack / budget / team / regulatory]
Failure modes:[what must never happen vs acceptable degradation]
Deliver:      [trade-off analysis / component diagram / sequence diagram /
               migration plan / capacity math]
Don't:        [patterns or services off the table]
```

### The Two Questions to Ask Every Design
```
"1. What's the simplest version of this that works at current scale?
    Build that first.

 2. What's the first thing that breaks when we 10x?
    Make sure we can fix that without a rewrite."
```

---

## 9. Architecture Decision Records

### Draft an ADR From a Conversation
```
"We just decided to use Kafka instead of RabbitMQ for our
 event pipeline. Here's why:
 - RabbitMQ loses messages on consumer crash without acking
 - We need replay capability for the audit log
 - Team already has Kafka in another service
 - Downside: ops complexity, slower local dev

 Write an ADR for this decision in the Nygard format."
```

### ADR From First Principles
```
"Help me write an ADR for our database choice for the new
 analytics service.

 Facts: [your facts]

 Don't write the ADR yet. First ask me the questions you need
 answered to write a complete, defensible ADR."
```

### Stress-Test a Draft ADR
```
"Here's an ADR draft. Attack it before I circulate it to the team.

 [paste draft]

 Look for:
 - Consequences that are understated or missing entirely
 - Alternatives dismissed too quickly without evidence
 - Assumptions in the Context that may not hold in 12 months
 - Missing stakeholders who will be affected
 - Any place where 'we decided X' is stated without the actual reason

 Give me a list of gaps. I'll revise before sharing."
```

### ADR for a Rejected Option
```
"Write an ADR for a decision we almost made but rejected.

 We considered migrating from REST to GraphQL for our public API.
 After 3 weeks of investigation we decided not to.

 Reasons against: [your reasons]

 Write this as a rejected ADR. Future engineers need to understand
 why we said no, so they don't relitigate it."
```

### Superseding an Old ADR
```
"ADR-003 from 2021 chose MongoDB for our content store.
 We're now migrating to Postgres. Write ADR-031 that:
 - References and supersedes ADR-003
 - Explains what changed since 2021
 - Documents the migration path
 - Is honest that this is a reversal and why

 ADR-003:
 [paste old ADR]"
```

### ADR Review for Completeness
```
"Review these ADRs for quality before we add them to the repo.

 [paste ADRs]

 Check each for:
 □ Context explains the problem, not the solution
 □ Decision is a single clear sentence
 □ Consequences lists both positive AND negative
 □ Alternatives were actually considered (not just mentioned)
 □ Status is accurate
 □ Can a new engineer understand this without asking anyone?

 Output a table: ADR | passes | gaps"
```

### ADR Prompt Template
```
Decision:     [one sentence — what was chosen]
Context:      [what problem forced a decision, what constraints existed]
Alternatives: [what else was considered and why each was rejected]
Consequences: [what gets better, what gets worse, what's now required]
Status:       [proposed / accepted / deprecated / superseded by ADR-N]
Audience:     [future engineer joining the team with no context]
Tone:         [honest about downsides — not a justification document]
```

### The One Rule for ADRs
```
"Write this ADR so that a future engineer who disagrees with
 the decision can understand exactly what would need to be
 different for the alternative to have been the right choice.

 If they can't tell from the ADR when it would be appropriate
 to revisit this decision, the ADR isn't done."
```

---

## 10. Documentation Writing

### Specify the Reader, Not Just the Topic
```
❌ "write documentation for this function"

✅ "Write documentation for this function.

    Reader: a backend engineer joining the team who knows Python
    but has never seen our codebase. They're trying to understand
    when to use this vs the similar fetch_user() function.

    Cover:
    - What it does and what it returns
    - When to use it (and when NOT to)
    - The one gotcha: it returns stale data if cache_ttl is set
    - One realistic usage example

    No docstring novels — 15 lines max.

    [paste function]"
```

### README — Give Claude the Project's Job
```
"Write a README for this project.

 The project: a CLI tool that syncs Notion databases to Postgres.

 Reader: a developer evaluating whether to use this tool.
 They'll spend 60 seconds on the README before deciding.

 Must answer in order:
 1. What does it do? (one sentence)
 2. Why would I use this over alternatives?
 3. How do I install and run it in under 5 minutes?
 4. What are the limitations I'll hit?

 Do NOT include: badges, contributing guide, license section,
 long feature lists, or architecture diagrams."
```

### Docstrings — Match the Existing Style
```
"Add docstrings to all public functions in this module.

 Match the existing style in the codebase:
 - Google style docstrings (Args/Returns/Raises sections)
 - One-line summary that fits in 72 chars
 - Args documented only if non-obvious (skip self, skip *args)
 - Raises section only if the function raises something the caller
   must handle — not every possible exception

 Do NOT document:
 - Private functions (underscore-prefixed)
 - Functions where the name + types already tell the full story

 [paste module]"
```

### Runbook — Structured for 3am Incidents
```
"Write a runbook for this alert: 'payment_processing_error_rate > 5%'

 Format it for an oncall engineer who is:
 - Woken up at 3am
 - May not be familiar with this system
 - Needs to decide: page the team or handle alone

 Structure:
 1. What this alert means (one sentence)
 2. Immediate triage: 3 commands to run first and what to look for
 3. Common causes ranked by frequency, with fix for each
 4. Escalation criteria: when to wake someone else up
 5. Links: dashboard, logs, runbook for upstream dependencies

 No prose paragraphs — bullet points and commands only."
```

### Migration Guide
```
"Write a migration guide from v1 to v2 of our SDK.

 Breaking changes:
 [paste list of breaking changes]

 Audience: developer who has v1 working in production and needs
 to upgrade with minimal disruption.

 For each breaking change:
 1. What broke (exact method/field that changed)
 2. v1 code snippet (what they have now)
 3. v2 code snippet (what to change it to)
 4. Why we changed it (one sentence — helps them trust the change)

 End with: what they can safely ignore (non-breaking additions)."
```

### Review Existing Docs for Quality
```
"Review this documentation for quality.

 [paste docs]

 Flag:
 - Anything that's factually wrong or likely outdated
 - Sections that are too vague to act on
 - Missing information a new user would immediately need
 - Steps that are out of order or have hidden dependencies

 Output: a prioritized list of gaps. I'll fix them one by one."
```

### Documentation Prompt Template
```
Audience:    [who reads this — their role, what they know, what they don't]
Goal:        [what they need to be able to DO after reading]
Format:      [README / docstring / runbook / ADR / reference / guide]
Constraints: [length limit / style guide / what to exclude]
Tone:        [for engineers / for end users / for 3am oncall]
Source:      [paste code, old docs, or describe the system]
```

---

## 11. Git Workflows

### Commit Message Writing
```
"Write a commit message for this diff.

 [paste git diff]

 Follow our convention:
 - Subject: imperative mood, 50 chars max, no period
 - Body: explain WHY, not what (the diff shows what)
 - If it fixes a bug: one sentence on what the bug was
 - If it's a breaking change: start body with BREAKING CHANGE:

 Don't mention file names or line numbers in the subject —
 those are in the diff."
```

### Commit Message Review
```
"Review these commit messages for quality.

 [paste git log --oneline]

 Flag:
 - Messages that describe WHAT instead of WHY
 - Vague messages ('fix bug', 'update', 'wip') that won't help
   future git bisect or blame
 - Missing context on breaking changes
 - Incorrectly scoped commits (multiple unrelated changes in one)

 For each bad message, write a better version."
```

### Branch Strategy Design
```
"Design a branch strategy for our team.

 Context:
 - 8 engineers, 2-week sprints
 - Deploy to production daily
 - 3 environments: dev, staging, prod
 - Mobile app release every 2 weeks (can't deploy continuously)
 - Hotfix requirement: patch prod within 2 hours of a P0

 Evaluate: GitFlow vs trunk-based vs GitHub Flow.
 For each: show the branch diagram, daily workflow, and
 how a hotfix works.

 Recommend one. Include the one scenario where it breaks down."
```

### Git Bisect — Finding a Regression
```
"Help me use git bisect to find the commit that introduced this bug.

 Bug: user login returns 401 on valid credentials.
 Last known good: tag v2.3.1
 First known bad: current HEAD (v2.4.0, 47 commits ahead)

 Write me:
 1. The exact bisect commands to start the session
 2. A test script I can pass to 'git bisect run' to automate it
   (should exit 0 if good, 1 if bad, 125 to skip)
 3. What to do when bisect finds the culprit commit"
```

### Cleaning Up a Messy Branch
```
"I have a feature branch with 23 commits — WIP saves, typo fixes,
 and 'undo last change' commits mixed in.

 I want to squash this into 3 logical commits before merging:
 1. Add the data model changes
 2. Add the API endpoint
 3. Add the tests

 [paste git log --oneline for the branch]

 Write the interactive rebase commands to get there.
 Flag any commits that don't fit cleanly into the 3 groups
 so I can decide what to do with them."
```

### PR Description Writing
```
"Write a PR description for this change.

 [paste git diff --stat and key changed files]

 Our PR template requires:
 - Summary: what changed and why (not how)
 - Test plan: what to verify before merging
 - Screenshots: flag if UI changed (I'll add them)
 - Breaking changes: explicit callout if any

 Audience: the reviewer who needs to understand the intent
 quickly, and future engineers reading git history.
 Don't summarize every file changed — explain the decision."
```

### Merge vs Rebase Decision
```
"Should I merge or rebase this feature branch before opening a PR?

 Context:
 - Branch is 2 weeks old, 18 commits
 - Main has moved 60 commits ahead
 - 3 merge conflicts in db/models.py
 - This is a public fork — other developers may have branched off mine

 Give me the trade-offs for this specific situation, then a recommendation.
 Include the exact commands for whichever you recommend."
```

### Git Workflow Prompt Template
```
Situation:   [what state the repo is in]
Goal:        [what you're trying to achieve]
Constraints: [shared branch? published commits? CI checks?]
Team:        [solo / small team / open source — affects risk tolerance]
Deliver:     [commands / strategy / decision]
```

---

## 12. Security Reviews

### Threat Model Before Code
```
"Before reviewing the code, build a threat model.

 System: user-facing file upload endpoint that stores files in S3
 and records metadata in Postgres.

 Enumerate:
 1. What can an attacker control? (inputs, headers, timing, volume)
 2. What's the worst outcome? (data exfiltration / RCE / DoS / account takeover)
 3. Who are the likely attackers? (external / authenticated user / insider)

 After the threat model, review this code against it:

 [paste code]"
```

### OWASP Top 10 Checklist Review
```
"Review this endpoint against the OWASP Top 10.

 [paste endpoint code]

 For each relevant category, tell me:
 - Vulnerable: yes / no / maybe
 - If yes: the exact attack vector and a proof-of-concept payload
 - Fix: minimal code change

 Categories to check:
 A01 Broken Access Control
 A02 Cryptographic Failures
 A03 Injection (SQL, command, LDAP)
 A04 Insecure Design
 A05 Security Misconfiguration
 A07 Authentication Failures
 A08 Software and Data Integrity Failures

 Skip categories that clearly don't apply."
```

### Secrets and Credential Audit
```
"Audit this codebase for secrets and credential mishandling.

 [paste files or describe structure]

 Look for:
 - Hardcoded secrets, API keys, passwords, tokens
 - Secrets passed via environment variables but logged
 - Credentials in config files that might be committed
 - JWT secrets that are too short or reused across environments
 - Database connection strings with credentials in the URL

 For each finding: file, line, risk level, and fix."
```

### Auth Flow Review
```
"Review this authentication flow for vulnerabilities.

 [paste auth code — login, token generation, session management]

 Check specifically:
 - Token entropy: is the secret long enough?
 - Token storage: where does the client store it and is that safe?
 - Expiry: are tokens short-lived? Is refresh handled securely?
 - Timing attacks: does login leak whether the user exists?
 - Brute force: is there rate limiting on login attempts?
 - Logout: does it actually invalidate the token server-side?

 Show the attack for each vulnerability you find."
```

### Dependency Vulnerability Scan Interpretation
```
"Here's the output of 'npm audit' / 'pip-audit' / 'trivy'.

 [paste audit output]

 For each vulnerability:
 1. Is it actually exploitable given how we use the package?
    (many are theoretical — help me triage)
 2. If exploitable: what's the attack scenario in our context?
 3. Fix: upgrade path, patch, or workaround?

 Prioritize by: exploitability in our stack × severity.
 I want a short list of what to fix this week, not a full audit."
```

### Input Validation Review
```
"Review all input validation in this module.

 [paste code]

 For every place external input enters the system:
 - Is it validated before use?
 - Is validation on the right side of any trust boundary?
 - Could validation be bypassed with type coercion, encoding,
   or unexpected content types?
 - Is error handling safe (doesn't leak internals in error messages)?

 Show me the specific bypass for each gap you find."
```

### Security Review Prompt Template
```
System:      [what the code does, who uses it, what data it handles]
Trust model: [who is authenticated / what can anonymous users do]
Worst case:  [what breach would be catastrophic vs acceptable]
Review for:  [specific OWASP categories / auth / injection / secrets]
Context:     [public internet / internal only / privileged users only]
Code:        [paste]
```

---

## 13. Incident Post-Mortems

### Build the Timeline First
```
"Help me reconstruct the incident timeline from these sources.

 Datadog alerts log:
 [paste]

 PagerDuty incident log:
 [paste]

 Slack thread (oncall channel):
 [paste]

 Deployment history:
 [paste]

 Build a chronological timeline with:
 - UTC timestamps
 - What was observed vs what was done
 - Who was involved at each step
 - Duration of each phase (detection / response / mitigation / resolution)

 Flag any gaps where we don't know what happened."
```

### Root Cause Analysis
```
"Help me write the root cause analysis for this incident.

 Incident: payment service returned 500s for 23 minutes.
 Timeline: [paste]
 What we found: [paste — logs, metrics, code]

 Use the 5 Whys method. For each Why, give me:
 - The answer based on evidence (not speculation)
 - The evidence that supports it
 - The next Why question

 Stop when you reach a systemic cause, not a proximate one.
 The goal is 'our on-call process didn't catch X' not 'engineer made a mistake'."
```

### Action Items — Specific and Assignable
```
"Turn these post-mortem findings into action items.

 Findings:
 [paste root causes and contributing factors]

 Rules for each action item:
 - Specific enough that someone can start it tomorrow
 - Has a single owner role (not 'the team')
 - Has a clear definition of done
 - Categorized as: prevent recurrence / improve detection / reduce MTTR
 - Estimated effort: hours / days / weeks

 Do NOT write vague items like 'improve monitoring' or 'add more tests'.
 Write 'add alert for payment_error_rate > 1% sustained 5 minutes'."
```

### Post-Mortem Write-Up
```
"Write the post-mortem for this incident.

 Audience: the whole engineering org — including people who
 weren't involved and won't know the context.

 Required sections:
 1. Summary (3 sentences: what happened, impact, duration)
 2. Timeline (chronological, UTC)
 3. Root cause (what actually caused it)
 4. Contributing factors (what made it worse or harder to detect)
 5. What went well (genuine — not performative)
 6. Action items (specific, owned, categorized)

 Tone: blameless. No 'X forgot to' or 'Y should have'.
 Frame around systems and processes, not people.

 Raw notes:
 [paste timeline, Slack threads, findings]"
```

### Detecting Patterns Across Incidents
```
"Here are our last 10 post-mortems.

 [paste post-mortem summaries or action item lists]

 Identify:
 1. Recurring root causes (same underlying issue appearing multiple times)
 2. Action items that were written but never fixed the problem
 3. Systems or components that appear in multiple incidents
 4. Detection gaps that keep showing up

 Give me the top 3 systemic issues to fix, ranked by
 frequency × blast radius."
```

### Post-Mortem Prompt Template
```
Incident:    [what broke, when, for how long]
Impact:      [users affected / revenue / SLA breach]
Sources:     [alerts / logs / Slack / deploy history to paste]
Deliver:     [timeline / RCA / action items / full write-up]
Tone:        [blameless — always]
Audience:    [oncall team / full eng org / exec summary]
```

---

## 14. Explaining Code

### Explain to a Specific Level
```
❌ "explain this code"

✅ "Explain this code to me.

    My background: I've been writing Python for 5 years but I've
    never used asyncio. I understand threading but not event loops.

    Don't explain basic Python syntax.
    Do explain: why async/await is used here instead of threads,
    what 'await' actually does to execution flow, and what would
    break if I removed the async keywords.

    [paste code]"
```

### Build a Mental Model, Not a Line-by-Line Walk
```
"Don't walk through this line by line. Instead:

 1. What is this code's job in one sentence?
 2. What's the key data structure and how does it change over time?
 3. What are the 2-3 most important decisions made in the implementation?
 4. What would go wrong if I changed X? (X = the thing that looks weird)

 [paste code]"
```

### Explain a Design Pattern in Context
```
"This code uses the observer pattern. I know what the observer
 pattern is abstractly, but explain how it's implemented HERE:

 - What is the subject and what are the observers in this codebase?
 - Why was this pattern chosen over a direct function call?
 - What's the cost of this abstraction in this specific case?
 - If I wanted to add a new observer, what exactly would I write?

 [paste code]"
```

### Explain Why, Not What
```
"I can read what this code does. Explain why it's written this way.

 Specifically:
 - Why does the cache check happen before the DB query but after
   the permission check? (line 34 vs line 41 ordering)
 - Why is the retry count hardcoded to 3 instead of configurable?
 - Why does this use a class instead of a module-level function?

 If you don't know and it's not obvious from the code, say so —
 don't invent a reason.

 [paste code]"
```

### Explain a Codebase for Onboarding
```
"I'm joining this codebase next week. Give me an orientation.

 [paste directory structure or key files]

 Tell me:
 1. The 5 files I should read first and why
 2. The 3 concepts I must understand before touching anything
 3. The one area that's known to be messy / has hidden complexity
 4. What a typical change looks like end-to-end
    (where does it start, what does it touch, how does it ship?)

 Don't describe every file — orient me so I can find things myself."
```

### Explain a Bug You Fixed
```
"Help me write an explanation of this bug for the team.

 Bug: [describe]
 Fix: [paste diff]

 Write two versions:
 1. Slack message (3 sentences, non-technical, for #engineering)
 2. PR description (technical, for the reviewer and future git blame)

 For version 2: explain the root cause, not just what line changed.
 Future engineers reading git blame need to understand why this
 was wrong, not just that it was."
```

### Explain Performance Characteristics
```
"Explain the performance characteristics of this code to me.

 I need to understand:
 - Time complexity as a function of input size
 - Where allocations happen and whether they're per-call or amortized
 - Which operations are O(1) vs O(n) as the dataset grows
 - At what input size this will start to feel slow

 Don't rewrite it — just explain it so I can decide if it's
 fit for purpose at our scale (inputs up to 500k items).

 [paste code]"
```

### Explain Prompt Template
```
My background: [what I know / what I don't]
Explain:       [what specifically to cover]
Skip:          [what I already understand]
Goal:          [what I need to be able to DO after the explanation]
Format:        [narrative / bullet points / analogy / diagram in ASCII]
Code:          [paste]
```

---

## 15. Database Migrations

### Design a Zero-Downtime Migration
```
"Design a zero-downtime migration to add a NOT NULL column
 to a table with 80M rows in production.

 Table: orders
 New column: confirmed_at TIMESTAMP NOT NULL
 Default value: created_at (backfill from existing column)
 Postgres 15, live traffic, can't lock the table

 Give me the exact migration steps in order, including:
 - Which steps can run with the table live
 - Where the dual-write phase is needed
 - How long each step will take (rough estimate)
 - The rollback procedure for each step
 - How I verify data integrity between steps"
```

### Rename a Column Without Downtime
```
"Design the migration to rename 'user_id' to 'account_id'
 across our schema.

 Context:
 - Column exists in 6 tables
 - Referenced by foreign keys in 3 tables
 - Used in 40+ queries across 8 services
 - Can't take downtime — 3 regions, 24/7

 I know the naive approach (ALTER TABLE RENAME COLUMN) locks the table.
 Design the expand-contract pattern for this rename.
 Show the phases: what runs in each deploy, what gets cleaned up last."
```

### Migration Review — Risk Assessment
```
"Review this migration before I run it in production.

 [paste migration file]

 Flag:
 - Any operation that takes a full table lock
 - Anything that's irreversible without a backup restore
 - Missing index that will cause slow queries after migration
 - Data loss risk (dropped columns with data, truncates, etc.)
 - Constraint additions that will fail if data violates them

 For each risk: severity, whether it's blocking, and the mitigation."
```

### Backfill Strategy for Large Tables
```
"Design a backfill strategy for this migration.

 Need to populate: orders.tax_amount (new column)
 Calculation: order_total * tax_rate (both columns exist)
 Table size: 200M rows, 15GB
 Traffic: write-heavy during business hours (9am-6pm UTC)

 Requirements:
 - Must not lock the table
 - Must not saturate the DB during business hours
 - Must be resumable if it fails halfway
 - Must be verifiable (I can check progress and correctness)

 Show the backfill script and the rate-limiting approach."
```

### Index Migration
```
"I need to add a composite index to a 50M row table in production.

 Table: events
 Index: (tenant_id, event_type, created_at DESC)
 Current: no index on these columns
 Postgres, live traffic

 Design the migration:
 1. How to build the index without locking (CONCURRENTLY)
 2. How long it will take (rough estimate given table size)
 3. What happens to queries during the build
 4. How to verify the index is being used after creation
 5. How to drop it safely if it causes problems"
```

### Rollback Design
```
"Design the rollback strategy for this migration.

 [paste migration]

 For each change, tell me:
 - Is it reversible? (some changes lose data permanently)
 - If reversible: exact SQL to undo it
 - If irreversible: what we need to do before running this migration
   (backup, dual-write, etc.) to make recovery possible
 - How long rollback would take given our table sizes"
```

### Migration Sequencing Across Services
```
"We're changing the orders table schema and 3 services read from it.
 Design the deployment sequence.

 Schema change: splitting 'address' (one column) into
 'address_line1', 'address_line2', 'city', 'zip' (four columns)

 Services:
 - order-service: writes orders (owns the schema)
 - fulfillment-service: reads address for shipping
 - reporting-service: reads address for invoices

 Design the expand-contract sequence:
 - What gets deployed first, second, third
 - How long the transition period is
 - How we handle reads/writes during the transition
 - When the old column gets dropped"
```

### Migration Prompt Template
```
Change:      [what schema change is needed]
Table:       [size / write volume / read volume]
Constraints: [no downtime / no table locks / reversible]
Services:    [what reads/writes this table]
Deliver:     [migration steps / rollback plan / backfill script / risk review]
```

---

## 16. Dependency Upgrades

### Breaking Change Analysis Before Upgrading
```
"I'm upgrading Django from 3.2 to 4.2. Analyze the breaking changes.

 [paste Django 3.2 → 4.2 release notes, or describe the upgrade]

 For each breaking change:
 1. Does it affect us? Grep for: [relevant patterns]
 2. If yes: what breaks and where
 3. Migration effort: hours / days / weeks

 Output a table I can use to plan the upgrade sprint.
 Prioritize by: affects us (yes/no) × effort to fix."
```

### Upgrade Impact on Our Codebase
```
"Scan our codebase for usage of these deprecated APIs being
 removed in the next major version.

 Deprecated APIs:
 [paste list from migration guide]

 Our codebase:
 [paste relevant files, or describe structure]

 For each deprecated usage found:
 - File and line
 - What it needs to change to
 - Whether the replacement is a drop-in or requires logic changes"
```

### Dependency Audit — Why Is This Here?
```
"Audit our package.json / requirements.txt for unnecessary dependencies.

 [paste dependency file]

 For each dependency:
 1. What does it do?
 2. Is it used directly or only a transitive dependency?
 3. Could it be replaced with a native API in our runtime version?
 4. Is it maintained? (flag anything with no release in 2+ years)

 Prioritize: flag the top 5 candidates for removal and why."
```

### Upgrade Sequencing for a Dependency Graph
```
"I need to upgrade package A from v2 to v4, but:
 - Package B requires A v2 or v3
 - Package C requires B v1, but B v2 supports A v4
 - We use both B and C

 Map the upgrade sequence. For each step tell me:
 - What to upgrade
 - What to test after that step
 - Whether we're in a broken state between steps (risk window)"
```

### Security Patch — Fast Path
```
"There's a CVE in [package] v[X]. We're on v[Y]. 

 CVE details: [paste]

 Tell me:
 1. Are we actually vulnerable? (given how we use the package)
 2. If yes: what's the attack scenario in our context?
 3. Fastest fix: patch version available? Workaround?
 4. What to test after patching to confirm nothing broke
 5. Is this urgent (patch today) or can it go in next sprint?"
```

### Lock File Conflict Resolution
```
"I have a lock file conflict after merging main into my branch.

 [paste conflicted section of package-lock.json / poetry.lock]

 Tell me:
 - What caused the conflict (two branches installed different versions)
 - Which version to keep and why
 - Whether I can just regenerate the lock file safely
 - What to check after resolving to make sure nothing regressed"
```

### Evaluating a New Dependency
```
"I'm considering adding [package] to our stack. Evaluate it.

 What we'd use it for: [describe use case]
 Alternatives I've considered: [list]

 Evaluate on:
 - Maintenance health (release frequency, open issues, contributors)
 - Bundle size / runtime overhead
 - Security track record (past CVEs)
 - API stability (breaking changes between major versions)
 - Whether native APIs or a smaller package could replace it

 Recommend: add it / use an alternative / build it ourselves.
 One paragraph justification."
```

### Dependency Upgrade Prompt Template
```
Current:     [package + version you're on]
Target:      [version you're upgrading to]
Why:         [security / feature / EOL / forced by another dep]
Codebase:    [size / how heavily the package is used]
Deliver:     [breaking change analysis / upgrade steps / impact scan]
Risk:        [can we take downtime? / is this prod-critical?]
```

---

## Quick Reference Cheat Sheet

### The Universal Upgrade: Bad → Good

| Instead of... | Say... |
|---|---|
| "fix this" | "fix only X — don't change Y or Z" |
| "optimize this" | "target: p99 < 200ms. profiler shows bottleneck at line 88" |
| "review this" | "review for security only — ignore style" |
| "refactor this" | "refactor internals — public API must not change" |
| "write tests" | "cover these 3 known bugs. no mocks. pytest." |
| "explain this" | "explain WHY it's written this way, not what it does" |
| "that's wrong" | "wrong because: [specific reason]. fix only that assumption" |
| "make it better" | "reduce cyclomatic complexity to under 10. early returns only" |

---

### Prompt Templates at a Glance

**Debugging**
```
Error: [exact text] | When: [always/intermittent/under load]
Changed: [recent deploy/config] | Hypothesis: [your guess]
Want: [diagnosis / fix / instrumentation]
```

**Performance**
```
Current: [p50/p99 metric] | Target: [specific number]
Profiler: [where time goes] | Constraints: [no schema changes / etc]
```

**Code Review**
```
Context: [what it does] | Risk areas: [auth/concurrency/etc]
Review for: [security/regression/API] | Skip: [style/naming]
Format: [BLOCKING/IMPORTANT/MINOR tiers]
```

**Refactoring**
```
Goal: [problem it solves] | Must not change: [API/exceptions/side effects]
Approach: [pattern/technique] | Out of scope: [explicit list]
```

**Testing**
```
Known bugs: [past failures to cover] | Strategy: [unit/integration/e2e]
Mock only: [exact list] | Validate: [break it on purpose after]
```

**API Design**
```
Consumers: [who + what they need] | SLA: [latency/availability]
Deliver: [endpoint list / spec / trade-offs] | Don't: [off-limits patterns]
```

**System Design**
```
Numbers: [current scale → target scale] | SLA: [what must never fail]
Constraints: [stack/budget/team] | Deliver: [diagram/trade-offs/capacity math]
```

**ADR**
```
Decision: [one sentence] | Context: [what forced the choice]
Alternatives: [what was rejected + why] | Consequences: [good AND bad]
```

**Documentation**
```
Audience: [role + what they know] | Goal: [what they can DO after]
Format: [README/runbook/docstring] | Tone: [engineer/user/3am oncall]
```

**Migration**
```
Change: [what schema change] | Table: [size/write volume]
Constraints: [no downtime/no locks] | Deliver: [steps/rollback/backfill]
```

---

### The 10 Highest-Leverage Habits

1. **Constraints before goals** — say what must NOT change before what you want
2. **Paste errors verbatim** — never paraphrase stack traces or output
3. **Give measurements** — p99, row counts, profiler output beat "it's slow"
4. **State your hypothesis** — even a wrong one saves 2-3 turns
5. **One job per turn** — compound asks dilute quality
6. **Lock the public contract** before any refactor or optimization
7. **Plan before execute** — "list the files first, I'll confirm before you proceed"
8. **Point at the specific error** when correcting — not "try again"
9. **Validate with a break** — make Claude prove tests fail on buggy code
10. **Front-load session invariants** — state constraints once, apply everywhere

---

### Anti-Patterns to Avoid

| Pattern | Why it fails |
|---|---|
| "make this better / cleaner / faster" | No target — Claude picks its own definition |
| Pasting the whole file for a one-line fix | Claude edits things you didn't intend |
| Accepting the first solution | Ask for trade-offs before committing |
| "that's wrong, try again" | Claude regenerates the correct parts too |
| Skipping profiler output | Claude guesses at the bottleneck |
| No mock boundary specified | Claude mocks too deep, tests prove nothing |
| Docs without a named reader | Claude writes for an imaginary average |
| ADR without downsides | Reads as a press release, not a decision record |
| System design without numbers | Claude designs for imaginary scale |
| Migration without rollback plan | You'll need it at 2am |

---

### Built-in Claude Code Commands

```bash
/review           # code review of current branch diff
/security-review  # security-focused pass on changed files
/ultrareview      # deep multi-agent review (billed — use before merge)
/init             # generate CLAUDE.md for the repo
/clear            # reset context when switching tasks
/config           # change model, theme, settings
```

**In-session shell:**
```
! <command>       # run a shell command and pipe output into the conversation
                  # e.g.: ! npm test, ! gcloud auth login, ! git log --oneline
```

---

### The One Meta-Rule

> **Tell Claude what success looks like, not just what to do.**
>
> "Optimize this query" → Claude decides what 'optimized' means.  
> "This query must return in under 50ms for a 10M row table — optimize it" → Claude optimizes toward your constraint.
>
> Every prompt in this guide is a variation of that rule applied to a specific domain.