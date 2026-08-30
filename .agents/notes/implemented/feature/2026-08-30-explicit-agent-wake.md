# Agent Note: Explicit Agent wake for pending inbox work

Status: implemented

English | [中文](2026-08-30-explicit-agent-wake.zh.md)

## Problem

Durable inbox input can survive process loss and reappear when a Session is resumed. `resume()` reconstructs that input but deliberately does not start a model turn. A recovery owner otherwise has to insert a second waking message, which changes the canonical audit log and can duplicate the persisted task intent.

## Decision

The public `Agent` interface requires `wake(): void`. It requests driver work for input already present in the inbox and never inserts, removes, or rewrites a message. An empty inbox is a no-op.

An idle agent starts its driver synchronously when pending input exists. A live driver already owns pending work. A wake behind maintenance or an actively cancelled operation latches until that activity converges, except disposal does not arm later work. Callers use `whenIdle()` to observe the resulting whole-agent activity rather than treating `wake()` as one message's result.

`resume()` remains inert after reconstructing a persisted inbox. The recovery owner inspects its durable evidence and calls `wake()` only when executing that exact pending input is safe.

## Alternatives considered

**Automatically drive every resumed inbox.** Resume is also used to inspect or continue Sessions whose injected context intentionally waits for later input. Automatic execution would turn persistence loading into an external-side-effect boundary and remove the recovery owner's decision point.

**Insert a synthetic follow-up or reinsert the persisted message.** Either choice adds another durable intent record and can duplicate task delivery. A wake operation must preserve the existing message identity and audit history.

**Expose a concrete loop helper.** Consumers would depend on `dsh-agent-loop` internals and make the public `Agent` implementation non-swappable. The operation belongs on the interface implemented by every driver.

## Consequences

Process-recovery consumers can resume and execute one canonical pending message without manufacturing input. Empty or inspection-only resumes remain inert. Every `Agent` implementation and exact test double implements the required method, while no Session event or model-visible text is added merely by calling it.
