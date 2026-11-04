# Spec‑Driven, Role‑Isolated Copilot Agent Workflow with Task Orchestration & Persistent Memory

> Use case (condensed) — internal proposal + open questions for the Copilot team.

## Problem

We operate a mix of monorepos and standalone repos, including many legacy and modern services lacking formal specs. Without a single source of truth, Copilot cannot safely fully automate code changes, test creation, impact analysis and code review. Additionally, chained agents suffer from context leakage, and every new task repeats prior research because there is no persistent, retrievable memory.

## Proposed Solution

We are building an internal workflow:

- A **Spec Agent** generates versioned specs from code and Git history; humans review and approve the spec, which becomes the only binding source of truth for downstream agents. Spec drafts are hidden from later stages.
- A **task dashboard** lets humans or agents create tasks. A human must move a task to **Ready** before our orchestrator spins up agents: either long‑lived dedicated functional agents (Spec / Coding / Testing / Review) or short‑lived generic agents.
- Each agent runs with **strict role‑based context isolation** to avoid confirmation bias and information leakage.
- A **two‑tier memory store** persists lessons: repo/module local memory and read‑only global org memory. After task completion, key decisions are distilled into memory; new agents retrieve role‑filtered memory before execution.

## Expected Outcome

Humans focus mostly on spec validation and the Ready gate. All mechanical engineering work runs on Copilot agents. Isolation improves review/test reliability, and scoped memory accumulates institutional knowledge across our repo landscape.

## Questions for Copilot Team

1. How to enforce bounded, role‑isolated context across chained specialized agents?
2. Recommended patterns for role‑filtered persistent memory across monorepos and multiple repos, plus stale‑memory invalidation?
3. How to lock downstream agents strictly to approved specs while hiding speculative drafts?
4. Can Copilot Agents API support both dedicated agent pools and externally triggered ephemeral agents?
