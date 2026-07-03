---
title: Interview Prep
tags:
  - interview
  - backend
  - golang
  - career
aliases:
  - Brief question
  - Skooldio Video Interview
  - INIT
---

# Interview Prep

## Snapshot
- Recent CS graduate from KU.
- Internship at Central Food Retail, working on Go Wholesale backend.
- Built backend services with Golang, Postgres, microservices, and GCP.
- Comfortable translating business problems into service design, API structure, and execution steps.

## Tell Me About Yourself
- Hi, I'm Kritpiruch, but most people call me Amp.
- I recently graduated from Kasetsart University, majoring in Computer Science.
- During my internship at Central Food Retail, I worked on backend systems for Go Wholesale.
- I collaborated with product, QA, business, and frontend teams to ship features and support production.

## Strengths
- Strategic and structured thinking: I like to map the overall process first, then break it into steps.
- Systematic with creativity: I work well when the core structure is clear, and I can adapt when the problem changes.
- Empathetic team player: I listen carefully and try to make collaboration easier for the team.

## Weaknesses
- I can miss small execution details if I do not keep a checklist.
- Public speaking and self-promotion are not my strongest areas, so I need to manage energy and pacing well.
- I can be a people pleaser and sometimes take on too much before asking for help early enough.

## Motivation Scripts

### Why Software Engineer
- Software engineering forces me to think across business logic, system design, reliability, and user impact.
- I like backend and infrastructure work because it lets me build a solid foundation that can evolve.
- The role matches how I naturally work: broad thinking first, then careful implementation.

### Why Skooldio
- I am interested in product tech that helps people upskill more easily.
- I expect a place like this to help me grow both technically and in communication and collaboration.
- I want work that combines product thinking, problem solving, and learning from a strong team.

### What Value Can You Bring
- I can connect business context with technical implementation.
- I work well with multiple teams and can keep communication focused on the actual problem.
- I tend to think ahead about readability, maintainability, and future changes in business logic.

### Why Should We Hire You
- I am open to feedback and different working styles.
- I do not want to just finish the assigned task; I try to make the solution easy for the team to extend later.
- I think about how to build code that is readable, reviewable, and safe for future changes.

## Go Wholesale Deep Dive

### Platform Overview
- Built a channel API that aggregates data from product, payment, orders, and other services.
- Worked with frontend teams and CMS portal users.
- Tech stack: Golang, Google Cloud, Postgres, Firestore, Pub/Sub, Cloud Tasks, Cloud Run.

### Cloud Platform Notes
- Cloud Run: request-driven serverless services.
- Cloud Run Jobs: task-driven jobs for batch processing.
- Pub/Sub: many-to-many messaging for background processing and fan-out.
- Cloud Tasks: one-to-one task scheduling for specific endpoints.
- Eventarc: event routing for changes such as file uploads.
- Cloud Tasks can be scheduled in advance, but the important thing in interviews is explaining idempotency and retry safety.

### Notification Scheduler
- CMS creates notification schedules with name, description, dates, sent time, recurrence, selected dates, and audiences.
- Notification setup service validates requests and writes schedule data to Postgres.
- Cloud Run Jobs run every midnight to find notifications due for the day.
- Each notification is scheduled into Cloud Tasks with the send time.
- Cloud Task calls the send-notification API, which toggles sent state and publishes a message to Pub/Sub.
- Notification process service consumes Pub/Sub, resolves audience tokens, and sends the notification through FCM.

### Notification for Go Bonus Targeted
- This is an extension of the notification scheduler.
- It targets customers in a Go Bonus campaign based on the remaining purchase amount before reaching the next tier.
- The main complexity is the heavy query needed to find the right customers.

### Go Bonus
- Tier-based reward campaign where customers accumulate purchase amounts to unlock coupon rewards.
- CMS is used to configure the campaign, tiers, and coupons.
- One campaign can have up to 50 tiers, and each tier can have coupon rewards.
- The feature is displayed in the app as a page and widget, along with promoted products.
- Purchase accumulation updates after an order is created.

### Order History
- Built order history support that could later be reused for reorder flows.
- Aggregated data from order history, product categories, product details, and cart data.
- Extracted unique SKUs before fetching product details.
- Used goroutines for batch retrieval of product details.
- Mapped product details back to each order and grouped results by order or category.
- A key challenge was avoiding nested loops and syncing concurrent results cleanly.

### Go Mail
- Built personalized promotion features based on customer segments.
- Combined product setup from CMS with personalized product data from the data team.
- Supported segments such as IE, IA, and IR.
- Cached SKU data in Redis after the first visit of the day to reduce repeated preload calls.
- Used preload metadata to fetch product details for display.

### Product Badge
- Added product badge support to upstream APIs.
- Updated app pages such as search and product listing to show badges.
- Filtered some categories out, such as alcohol products.

## STAR And Process Notes

### Introduction Structure
- Situation: internship experience at Central Food Retail / Go Wholesale.
- Task: design and build backend services, APIs, and app features.
- Action: coordinate with business, plan features, test, and support production.
- Result: shipped features and handled production support after deployment.

### Agile And Delivery
- Two-week sprints.
- Planning and estimation through planning poker.
- Sprint grooming with business for next-sprint planning, system design, and API specs.
- Retrospective topics: what went well, what to improve, and appreciation.
- Deployment across DEV, SIT, UAT, and PROD.

### GCP Detail Bank
- Cloud Run is request driven, so it fits APIs.
- Cloud Run Jobs work well for scheduled or batch tasks.
- Pub/Sub is useful when multiple services need to react to one event.
- Cloud Tasks fits explicit task dispatch to a concrete endpoint.
- When discussing Cloud Tasks, mention scheduling, retries, and deduplication concerns.

## Technical Bank

### Go Fundamentals
- Thread: managed by the OS.
- Goroutine: managed by the Go runtime.
- Use `errgroup` to manage multiple goroutines with shared context.
- Mutex: use when goroutines access shared memory.
- Channel: use for passing data or coordination between goroutines.
- Race condition: avoid mutable shared state, use `sync.Mutex`, channels, atomics, and `go test -race`.
- Context: carry request-scoped values, cancellation, deadline, and timeout.

### Authentication And Authorization
- Authentication answers "who are you".
- Authorization answers "what are you allowed to do".
- JWT has header, payload, and signature.
- Access tokens are short-lived and used in API requests.
- Refresh tokens are long-lived and used to mint new access tokens.
- A common pattern is to keep the access token in memory and the refresh token in a cookie.

### OAuth2.0
- OAuth2 allows a third-party app to authenticate users without handling their password directly.
- Typical Google login flow:
  - Client redirects to Google authorization page.
  - Google redirects back with an authorization code.
  - Server exchanges the code for tokens.
  - Server receives access token, ID token, and refresh token.

### Git
- Merge combines feature work back into the target branch.
- Rebase replays commits on top of a new base branch.

### Interview Prompts
- Can this person solve engineering problems?
- Can this person explain the systems they built?
- Can this person work as a team?
- Does this person have a growth mindset?
- Can this person survive?

### Questions To Ask The Team
- For an early-career software engineer, what does growth look like in the first 1 to 2 years?
- What does onboarding look like when a junior joins the team?
- What are the success criteria for someone who is doing well?
- What is the team culture and sprint process like?
- What challenges is the team facing right now?
- How much is AI used in the workflow, and what do you expect from a junior role?

## Technical Questions From Raw Notes

### Go Questions
- Thread and goroutine.
- Mutex and channel.
- Race condition.
- Go context.

### Postgres And DB
- Review the main talking points for schema, querying, indexing, and transactions before interviews.

### Microservice
- Review how service boundaries, aggregation, and upstream/downstream integration were handled in Go Wholesale.

## Source Notes
- [[about-go-wholesale|About Go Wholesale]]
- [[brief-question|Brief question]]
- [[skooldio-video-interview|Skooldio Video Interview]]
- [[INIT]]
