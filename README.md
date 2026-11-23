# Web Locks LLM Wiki & Interactive Slides

A structured knowledge base (**LLM Wiki**) and zero-runtime-dependency interactive HTML presentation covering **Multi-Tab State Coordination**, the **Web Locks API (`navigator.locks`)**, and the **Request Ledger Pattern**.

> 🔗 **Live Interactive Presentation**: [https://win4son.github.io/web-locks-llm-wiki-slides/](https://win4son.github.io/web-locks-llm-wiki-slides/)

---

## Overview

Modern single-page applications (SPAs) often suffer from duplicate network requests, cache race conditions (`Lost Update`), and authentication token refresh collisions when users open multiple browser tabs simultaneously. 

This repository demonstrates a **grounded knowledge-to-presentation workflow**:
1. **Structured LLM Wiki (`wiki/`)**: A curated, schema-enforced Markdown knowledge base broken down into atomic entries (`WIKI-00` through `WIKI-06`), each with explicit scope boundaries, real-world analogies, and technical constraints.
2. **Grounded HTML Deck (`index.html`)**: A self-contained, single-file interactive slide deck generated strictly from the wiki entries, complete with keyboard navigation and a built-in **Wiki Provenance Toggle** that maps every slide back to its source document.

---

## Repository Structure

```text
web-locks-llm-wiki-slides/
├── index.html                        # Self-contained interactive tutorial slides (zero external CDN)
├── README.md                         # Project documentation
└── wiki/                             # Structured Markdown knowledge base
    ├── index.md                      # [WIKI-00] Index, reading path, and unified glossary
    ├── 01-multi-tab-problem.md       # [WIKI-01] Concurrency pitfalls & duplicate request overhead
    ├── 02-legacy-approaches.md       # [WIKI-02] Why localStorage polling & BroadcastChannel fall short
    ├── 03-web-locks-api.md           # [WIKI-03] Native navigator.locks & crash-safe lifecycle
    ├── 04-request-ledger-pattern.md  # [WIKI-04] Single-flight coordination via Double-Checked Locking
    ├── 05-failure-recovery.md        # [WIKI-05] Timeout handling (AbortSignal), ifAvailable & rollbacks
    └── 06-architecture-tradeoffs.md  # [WIKI-06] Client-side tab locks vs. server-side distributed locks