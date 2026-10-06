# Platform Architecture

This document describes the architecture of a large-scale web data collection platform that I worked with.

The original implementation consists of multiple services and repositories. The architecture below has been generalized to remove proprietary implementation details while preserving the engineering concepts and system relationships.

---

## 1. System Overview

The platform can broadly be divided into two major areas:

* **Orchestration & Discovery** — responsible for finding, processing, enriching, and organizing URLs.
* **Scraping & Processing Engine** — responsible for executing scraping jobs, handling providers, parsing responses, and producing normalized results.

These components communicate through APIs, queues, collections, and persistent storage.

```text
                         ┌──────────────────────┐
                         │    Data Sources      │
                         │  Database / Search   │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │  Discovery &         │
                         │  Orchestration       │
                         │                      │
                         │  - Input processing  │
                         │  - Sitemap discovery │
                         │  - URL discovery     │
                         │  - Crawl management  │
                         └──────────┬───────────┘
                                    │
                             URLs / Crawl Jobs
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │       Scraping Engine         │
                    │                               │
                    │  Queue → Assemble → Scrape    │
                    │          → Parse → Write      │
                    └──────────────┬────────────────┘
                                   │
                                   ▼
                    ┌───────────────────────────────┐
                    │       Provider Layer          │
                    │                               │
                    │ Provider A │ Provider B │ ... │
                    └──────────────┬────────────────┘
                                   │
                                   ▼
                              HTML / JSON
                                   │
                                   ▼
                    ┌───────────────────────────────┐
                    │     Extraction & Parsing      │
                    │                               │
                    │  Site handlers / pagination   │
                    │  embedded content / iframes   │
                    └──────────────┬────────────────┘
                                   │
                                   ▼
                    ┌───────────────────────────────┐
                    │      Data Processing           │
                    │                               │
                    │ Normalize → Validate → Store  │
                    └──────────────┬────────────────┘
                                   │
                         ┌─────────┴──────────┐
                         ▼                    ▼
                    ┌──────────┐        ┌──────────┐
                    │ MongoDB  │        │Snowflake │
                    └──────────┘        └──────────┘
```

The system is intentionally distributed. Scraping execution, orchestration, persistence, and model-assisted processing are separated so that each part can scale and fail independently.

---

# 2. Discovery & Orchestration

The orchestration side is responsible for turning an initial set of entities or websites into candidate URLs that can be processed by the scraping infrastructure.

A simplified flow is:

```text
Input Data
    │
    ▼
Input Filtering
    │
    ├──────────────► Sitemap Discovery
    │
    └──────────────► Homepage Discovery
                           │
                           ▼
                     Recursive Crawl
                           │
                           ▼
                    Candidate URLs
                           │
                           ▼
                    Validation / Merge
                           │
                           ▼
                       Storage
```

### Input processing

Initial input data is retrieved from a database and converted into internal objects used throughout the discovery pipeline.

Filtering is applied before expensive scraping work begins.

### Sitemap discovery

The system attempts to identify sitemap and robots.txt resources and expands sitemap structures when necessary.

Sitemap URLs can then be inspected to identify likely job-related pages.

### Homepage and recursive discovery

When sitemap discovery is insufficient, the platform can crawl websites starting from their homepage.

The crawl can discover:

* Links
* Forms
* Scripts
* Embedded content
* Iframes
* Pagination
* Additional candidate URLs

This creates a recursive discovery process where newly discovered URLs can lead to additional candidates.

---

# 3. Scraping Engine

The scraping engine is a queue-driven processing system.

Rather than fetching and processing a URL in one operation, work moves through several stages.

```text
                ┌──────────────┐
                │     Queue    │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │   Assembler  │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │    Scraper   │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │    Parser    │
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │    Writer    │
                └──────────────┘
```

Each stage has a specific responsibility.

### Assembler

The assembler prepares the request that needs to be executed.

It determines the appropriate request method and provider configuration before passing the job to the scraping stage.

### Scraper

The scraper executes the request against the selected provider.

The raw response can contain:

* HTML
* JSON
* Response headers
* Status codes
* Resolved URLs
* Provider-specific metadata

Raw responses are persisted so that downstream processing can occur independently of the original network request.

### Parser

The parser retrieves the raw response and determines how it should be processed.

It selects the appropriate website-specific handler and extracts the required information.

### Writer

The writer takes normalized results and writes them to the configured output.

Keeping writing separate from scraping and parsing means that individual stages can be retried without necessarily repeating the entire operation.

---

# 4. Provider Abstraction

One of the important architectural concepts is the abstraction between the scraping engine and external scraping providers.

```text
                    Generic Request
                          │
                          ▼
                ┌──────────────────┐
                │ Provider Adapter  │
                └────────┬─────────┘
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
         Provider A  Provider B  Provider C
             │           │           │
             └───────────┼───────────┘
                         ▼
                    Normalized
                     Response
```

The rest of the engine should not need to understand every provider's API.

Instead, provider adapters translate generic scraping requirements into provider-specific requests and convert provider responses back into a format understood by the engine.

This becomes particularly important when providers differ in how they support:

* JavaScript rendering
* Browser actions
* Cookies
* AJAX capture
* Iframe content
* Waiting strategies
* Geographic targeting
* Sessions
* Error reporting

---

# 5. Provider Selection & Fallbacks

Provider execution is separated from provider selection.

A flow can maintain a list of available providers and select an initial provider before execution.

When a request fails, the system can determine whether it should:

1. Retry the same provider
2. Switch to another provider
3. Re-enqueue the job
4. Treat the failure as terminal

```text
             Request
                │
                ▼
         Select Provider
                │
                ▼
          Execute Request
                │
          ┌─────┴─────┐
          │           │
       Success      Failure
          │           │
          ▼           ▼
       Continue    Retry / Rotate
                      │
                      ▼
                Next Provider
```

This abstraction allowed new providers to be introduced without redesigning the entire scraping pipeline.

---

# 6. JavaScript & Dynamic Websites

A significant challenge in web scraping is that the HTML returned by a normal HTTP request is not always the HTML visible to a user.

Some websites require:

* JavaScript execution
* Browser interaction
* Waiting for network activity
* AJAX requests
* Cookies generated during page execution
* Embedded content retrieval

The platform therefore supports different rendering and browser-related capabilities through the provider layer.

Conceptually:

```text
URL
 │
 ├── Static request ──────► HTML
 │
 └── Browser request
          │
          ├── JavaScript
          ├── Browser actions
          ├── Network activity
          ├── AJAX
          └── Embedded content
                    │
                    ▼
                  HTML
```

An important engineering challenge is that generic engine requirements do not always map directly to equivalent provider parameters.

Provider integrations therefore need to translate these requirements carefully.

---

# 7. Pagination

Pagination is handled as part of the scraping and parsing workflow rather than being treated as a simple loop.

A page can provide information about how the next page should be reached.

```text
Page 1
  │
  ▼
Extract Results
  │
  ▼
Detect Pagination
  │
  ├── No next page ──────► Finish
  │
  └── Next page
          │
          ▼
        Queue
          │
          ▼
        Page 2
          │
          ▼
        ...
```

The system also needs to prevent pagination loops.

Previous-page information and extracted results can be used to detect situations where the same content is repeatedly returned.

---

# 8. Embedded Content & Iframes

Some websites do not contain the useful data directly in the main document.

Instead, information may be provided through:

* Iframes
* Embedded JSON
* Client-side application state
* AJAX responses
* External widgets

The platform supports site-specific handlers for these cases.

This is important because there is no universal solution for every embedded application. The extraction strategy often depends on how the particular website exposes its data.

---

# 9. AI-Assisted Processing

A separate part of the platform uses an LLM to process difficult or ambiguous discovery cases.

The general flow is:

```text
                 Candidate URL
                       │
                       ▼
                  Fetch HTML
                       │
                       ▼
              Clean / Prepare HTML
                       │
                       ▼
               Custom Prompt
                       │
                       ▼
                  LLM API
                       │
                       ▼
             Structured Response
                       │
                       ▼
              Normalize / Validate
                       │
                       ▼
                    Storage
```

The model receives the page HTML together with a task-specific prompt.

The prompt defines the information and structure expected from the model.

The LLM is therefore used as one processing component inside a larger deterministic pipeline.

This approach is particularly useful when traditional selectors or heuristics cannot reliably determine the structure of a website.

---

# 10. Data & Persistence

Different storage systems serve different purposes within the platform.

A simplified representation is:

```text
Scraping
   │
   ▼
Raw Response
   │
   ▼
Staging Storage
   │
   ▼
Parsing
   │
   ▼
Normalized Result
   │
   ├──────────► Object/File Storage
   │
   ├──────────► Operational Database
   │
   └──────────► Analytical Database
```

Keeping raw responses separate from normalized results provides an important operational advantage.

A parsing or transformation problem can often be investigated or retried without making another external scraping request.

---

# 11. Concurrency & Queue Management

The platform performs large amounts of network work, so concurrency cannot simply be set to an arbitrary high value.

Concurrency is influenced by:

* Provider limits
* Queue depth
* Worker capacity
* Flow configuration
* Resource availability
* Retry pressure

A control layer monitors the system and manages queue and worker behavior.

```text
                 Controller
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Queues     Workers    Config
          │          │          │
          └──────────┼──────────┘
                     ▼
              Concurrency
                 Control
```

This allows scraping workloads to be scaled while respecting external provider and infrastructure constraints.

---

# 12. Reliability & Failure Handling

Because the platform depends heavily on external websites and third-party services, failure is expected.

Failures can originate from:

* Website changes
* HTTP errors
* Provider errors
* Rendering failures
* Timeouts
* Parsing failures
* Rate limits
* Temporary infrastructure issues

The system therefore uses retries, backoff, provider rotation, and failure classification.

The objective is not simply to retry everything.

A useful retry strategy needs to distinguish between:

```text
Transient failure
       │
       ▼
    Retry
       │
       ▼
   Recover?

   Yes ──► Continue

   No
       │
       ▼
 Try another provider
       │
       ▼
   Recover?

   Yes ──► Continue

   No ──► Terminal failure
```

---

# 13. Docker & Service Architecture

The platform is composed of multiple services rather than a single application process.

A simplified local environment looks like:

```text
                    Docker Environment
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
    Starter/API         Workers            Controller
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
         Assembler       Scraper        Parser
                                           │
                                           ▼
                                         Writer

             ┌─────────────────────────────────┐
             │ Redis + Database Services        │
             └─────────────────────────────────┘
```

Docker provides a reproducible environment for running and debugging the different services together.

---

# 14. CI/CD

The development workflow uses automated CI/CD infrastructure.

The repositories use GitHub Actions for repository-level testing and deployment automation, while Jenkins is also involved in operational delivery workflows.

The general workflow is:

```text
Git Change
    │
    ▼
Pull Request / Push
    │
    ▼
Automated Tests
    │
    ▼
Build
    │
    ▼
Container Image
    │
    ▼
Deployment / Delivery
```

This provides automated validation before changes reach downstream environments.

---

# 15. Architecture Summary

The most important architectural characteristics are:

### Distributed processing

The platform separates orchestration, scraping, parsing, writing, and persistence into different components.

### Queue-driven execution

Work moves through asynchronous stages, allowing individual parts of the pipeline to scale independently.

### Provider abstraction

External scraping providers are hidden behind a common interface, making it possible to introduce and evaluate different providers.

### Site-specific extraction

Different websites can require different extraction strategies, while still operating within the same overall scraping architecture.

### Failure-aware processing

Retries, provider rotation, backoff, and concurrency controls are built around the assumption that external dependencies will fail.

### AI-assisted processing

LLMs are used selectively for cases where deterministic scraping and discovery techniques are insufficient.

### Separation of concerns

The architecture separates:

```text
Discovery
    ↓
Orchestration
    ↓
Scraping
    ↓
Parsing
    ↓
Normalization
    ↓
Validation
    ↓
Persistence
```

This separation makes it possible to modify or replace individual parts of the platform without redesigning the entire system.

---

## What I Worked On

My work touched several layers of this architecture, including:

* Website scraper maintenance and development
* Scraping provider integration
* Provider benchmarking
* Dynamic rendering and provider capability investigation
* URL discovery and processing
* AI-assisted HTML processing
* Failure investigation and retry behavior
* Concurrency configuration
* Docker-based development
* Git workflows
* CI/CD workflows
* Scraping cost optimization

The detailed progression of these contributions is documented in [contributions.md](contributions.md).
