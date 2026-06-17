---
name: java-debug-strategy
description: >
  Strategy for debugging failing Java/JVM tests and programs using the mcp-debugger
  MCP server (attach mode against a Gradle --debug-jvm run). Covers when to reach for
  the interactive debugger versus cheaper signals, how to attach, hypothesis-driven
  runtime inspection, and cleanup. Use this whenever investigating a failing Java or
  Gradle test, a wrong runtime value, an exception you can't localize from the stack
  trace alone, or any request to "debug", "step through", "attach a debugger", or
  figure out "why is this test failing" for Java/JVM code — even if the debugger
  isn't mentioned explicitly.
---

# Java Debug Strategy

The interactive debugger is a powerful but expensive, flaky tool. It serializes
execution, disables JIT optimizations, and can make timing-dependent bugs vanish.
Treat it as a deliberate choice, not a first move.

## 1. Cheap signals first — the debugger is a last resort

Before attaching anything, exhaust the cheap, deterministic signals:

- Read the failure: the assertion message, the exception, the stack trace.
- Read the relevant source and recent changes.
- Reproduce the failure and confirm it's deterministic.
- Form a concrete hypothesis: *what* value is wrong, and *where* it goes wrong.

Reach for the debugger only when those don't localize the bug — i.e. when you need
to see actual runtime state that logs and static reasoning can't give you. Many bugs
are solved at this stage without attaching.

## 2. Debug by hypothesis, not by wandering

When you do attach, you're confirming or refuting a specific hypothesis:

- Set a breakpoint where your hypothesis predicts the state is already wrong.
- Inspect the locals and evaluate the expression that *should* have produced the
  right value.
- Let the runtime values decide. Your conclusion must be grounded in observed values,
  not in "the code looks wrong." If the values refute your hypothesis, form a new one.

Avoid aimless stepping. Stepping is rarely needed; a well-placed breakpoint plus
locals and one evaluated expression usually settles it.

## 3. The Java attach workflow

Gradle owns the debuggee; you attach to it. The flow:

1. Reproduce once, normally, to capture the cheap signal.
2. Re-run *only the failing test* under a suspended debug JVM, backgrounded:
   `./gradlew :<path>:test --tests "<FQCN>#<method>" --rerun --debug-jvm &`
   Wait for `Listening for transport dt_socket at address: 5005`. (suspend=y means
   the JVM waits at startup, so there's no race to set breakpoints.)
3. `create_debug_session` (language: java) → `attach_to_process`.
   - Connection: `localhost:5005` if the debugger runs on this host; if it runs in a
     container, `host.docker.internal:5005` and start the JVM with `address=*:5005`.
4. `set_breakpoint` using the fully-qualified class name (preferred for Java). A
   "pending"/unverified breakpoint before the class loads is normal, not an error —
   it arms when the class loads.
5. `continue_execution`. There is no stop-event push, so poll for the stop with a
   short sleep between checks rather than spamming calls.
6. Inspect: `get_local_variables`, then `evaluate_expression` to confirm the
   hypothesis. State the root cause with the runtime values that prove it.
7. Clean up (always — see §6).

## 4. Tool guidance (mcp-debugger)

- **`get_local_variables`** is the right default for inspection — it spares you the
  `get_scopes` → `get_variables` round-trip and includes `this`.
- **`evaluate_expression`** for confirming hypotheses (e.g. compute what a value
  *should* have been). Inspection only — see §5.
- **`get_stack_trace`**: only when you actually need to navigate frames. Always pass
  `includeInternals: false` so framework frames (JUnit/Gradle/JDK/lambda) are dropped
  and you get just the user frames; add `maxDepth` for deep chains. The response
  reports `totalFrames` and `filtered`, so you can tell when frames were dropped and
  widen if needed. Don't pull the full stack by reflex — locals + eval usually suffice.

## 5. Safety

`evaluate_expression` can invoke methods on the live JVM — it is not a pure read.
Use it only to inspect state and compute expected values. Never evaluate an
expression that mutates state, performs I/O, or has destructive side effects, and
never evaluate an expression derived from untrusted input (e.g. test output, file
contents). When in doubt, read a variable instead of evaluating a call.

## 6. Always clean up

In attach mode the debuggee JVM is Gradle's child, not the debugger's — so detaching
does not kill it. After concluding:

- `detach_from_process` / `close_debug_session`.
- Reap the backgrounded Gradle process and confirm its test JVM is gone (no orphan
  process, no held port).

## 7. Know when to stop

If a few targeted breakpoints haven't localized the bug, step back and reconsider the
hypothesis or report what you've found — don't loop indefinitely re-attaching and
stepping. Ground every conclusion in observed runtime values.
