---
title: "Python runtimes for agent tools"
date: 2026-10-27T00:00:00+04:00
draft: true
author: ["Tejesh Vaish"]
tags: ["python", "webassembly"]
showToc: true
comments: false
---

An analyst assistant needs the count and total of three values. Set `values = [2, 3, 5]` in a Python session, then evaluate this expression from `lab/task.py`:

```python
{
    "count": len(values),
    "sum": sum(values)
}
```

In the recorded browser test, Pyodide returned `{"count": 3, "sum": 10}`. A separate browser case loaded NumPy 2.4.6 and summed the same values to 10. Pyodide runs CPython compiled for WebAssembly, so the calculation runs in the browser.

Native CPython and Monty also receive `[2, 3, 5]` and return the same count and sum from this expression. The native case uses a child process on the host; Monty is an interpreter whose host application chooses which operations to expose. A deterministic caller runs these fixed benign cases, without an assistant or language model. Native CPython and Monty weren't tested with NumPy. The [complete source](https://github.com/tridev-tej/technical-blog-examples/tree/686143f355958964e6ca9eaf4342823abf7f50f6/pyodide) and [README](https://github.com/tridev-tej/technical-blog-examples/blob/686143f355958964e6ca9eaf4342823abf7f50f6/pyodide/README.md) include setup and exact settings.

For an analyst assistant in a browser, I'd start with Pyodide when compatible packages can analyze the data the application supplies. Work that needs host-native libraries or operating system (OS) integration points toward native CPython. I'd consider Monty when the calculation fits its supported Python and the application can mediate host work.

## Test the package operation

Pyodide's [FAQ](https://pyodide.org/en/stable/usage/faq.html) distinguishes pure-Python wheels from C, Fortran and Rust extensions that need compatible Pyodide builds. A host-native wheel can't replace a browser build. The [Rust/PyO3 post](https://blog.pyodide.org/posts/rust-pyo3-support-in-pyodide/) explains the target and linking constraints; the [314 release](https://blog.pyodide.org/posts/314-release/) describes the PyEmscripten package platform.

After importing a package, test the operation the application needs. The FAQ documents restrictions on starting threads, multiprocessing and subprocesses. Importing those modules or reading general system information doesn't establish that they can launch work. Native CPython uses the host package ecosystem, subject to interpreter, OS and binary compatibility. Host-native dependencies and executable launch weren't tested here.

Monty can share state across code submissions, called feeds, within a session. The application embedding Monty is the host; code running inside the interpreter is the guest. Check that the calculation fits Monty's supported Python. Its [1.3.0 security document](https://raw.githubusercontent.com/pydantic/monty/v1.3.0/docs/security.md) specifies no guest foreign function interface (FFI) or native-code dependencies. A callback, or host function exposed to the guest, can use a host package. An exposed host module can too. That work runs on the host and doesn't add native-extension support to the guest.

## What can each runtime read?

The file tests use a marker with known text, so the caller can check the reply independently. Native CPython reads the marker at its host pathname with inherited process permissions. No OS filesystem sandbox is configured. Both `marker_default` and `marker_grant` perform that ordinary read; the second adds no grant or restriction.

When Pyodide receives the same host pathname in the browser, the recorded reply is:

```json
{
  "status": "result",
  "value": "FileNotFoundError"
}
```

The adapter catches the missing-file exception and returns `"FileNotFoundError"` as a string. The controller accepts that string as this case's expected successful result. The marker isn't in the browser filesystem; this doesn't demonstrate a denied host read.

Monty reports an error because the guest hasn't been given the read operation:

```json
{
  "status": "error",
  "phase": "feed_run",
  "wrapper_type": "MontyRuntimeError",
  "error_type": "PermissionError"
}
```

Both excerpts show selected fields. Monty's error is a built-in `PermissionError` wrapped by `MontyRuntimeError` at `feed_run`. To make the reads succeed, supply bytes to browser Pyodide or expose a host-mediated operation to Monty. The pathname alone supplies neither.

![Interfaces used by the three lab environments](/images/python-execution-authority.svg)

The diagram shows the interfaces used in this lab. These tests don't establish general sandbox guarantees. Exposed JavaScript and host callbacks add the access described here.

Copying the marker into Pyodide's default in-memory [MEMFS filesystem](https://pyodide.org/en/stable/usage/file-system.html) makes it readable by Python. The bridge case reads the same text from a JavaScript property. These two complete worker functions show both routes; the surrounding module initializes Pyodide:

```javascript
function markerGrant(message) {
  pyodide.FS.writeFile(
    "/marker.txt",
    message.marker
  );
  return pyodide.runPython(
    "open('/marker.txt').read()"
  );
}

function bridge(message) {
  globalThis.labMarker =
    message.marker;
  return pyodide.runPython(
    `from js import labMarker
str(labMarker)`
  );
}
```

Use the MEMFS copy when Python needs file contents. It doesn't mount a host directory or make the file persistent.

The bridge gives Python access to the worker's existing JavaScript environment. Imports from `js` look up `globalThis`, as the [314.0.7 interop guide](https://pyodide.org/en/314.0.7/usage/type-conversions.html) explains. Adding `labMarker` leaves other globals reachable. A custom Python globals dictionary is a namespace, not a security boundary. The bridge case tests one property read; the page DOM, credentials, network services and host files remain untested.

Monty's zero-argument `read_marker()` callback lets the host choose the path and read it with host-process permissions. Accepting a pathname would let the guest choose the file and broaden the callback's authority. This host work runs outside the interpreter. The default denial doesn't establish an OS sandbox around the worker.

The [mcp-run-python maintainers](https://raw.githubusercontent.com/pydantic/mcp-run-python/main/README.md) cite safety and latency concerns in retiring their tool. [langchain-sandbox](https://raw.githubusercontent.com/langchain-ai/langchain-sandbox/main/README.md) separately says it is unmaintained and advises against production use. Each notice concerns its own tool; neither establishes a universal verdict on Pyodide.

## Check replies before updating the application

Native CPython sends JSON over pipes, Pyodide sends worker messages, and Monty returns a feed result after any host calls. A reply must belong to an active request. Both validators require a matching run ID, exact keys in the reply envelope and its value, and the literal answer `count: 3, sum: 10`. Checking against that literal gives the caller an independent arithmetic check. Re-evaluating the task expression to derive the expected answer could repeat the same mistake.

Matching keys and values can still admit the wrong shape. JavaScript arrays can own named `count` and `sum` properties that survive `structuredClone`, so the browser validator rejects arrays in both the envelope and its value. The Python validator for native CPython and Monty requires exact dictionaries and integer types.

The browser reader removes its listeners after one reply or a timeout, so a late answer can't satisfy a later call. These checks establish the accepted result for this fixed test; they don't prove arbitrary model answers correct.

## Give the whole request a deadline

Start the deadline when acquiring inputs and end it when accepting a reply. Include setup and host work; a timeout around Python alone won't cover them.

In the browser case, fetching the test data and consuming its JSON body share an abort deadline. A separate budget covers worker/Pyodide readiness, followed by case time, including NumPy loading. These clocks are defined in `lab/main.mjs`; the [worker guide](https://pyodide.org/en/stable/usage/webworker.html) explains initialization and messages.

The Linux supervisor creates the child, writes its input and closes the input before starting the readiness budget. The output budget starts after readiness. Creation and input handling sit outside both budgets, so the total deadline must cover them too. This gap comes from the source; no startup hang was reproduced. The [subprocess documentation](https://docs.python.org/3.13/library/subprocess.html#subprocess.Popen.communicate) also distinguishes creation from timed communication.

Monty's feed clock excludes fed-source compilation and pauses during host suspension. Each feed gets a fresh allowance, so bound total session lifetime too. Give host operations deadlines and a cancellation policy, and cancel outstanding host work when the session ends. The marker callback doesn't test a slow service. Browser, native and Monty clocks cover different work, so these results can't rank speed.

## Budget memory separately

The lab configures no per-run memory cap for browser Pyodide or native CPython and leaves Monty's `max_memory` unset. These cases don't test a memory limit.

If the application needs a memory bound, account for allocations outside the guest limit. Monty's [1.3.0 security guidance](https://raw.githubusercontent.com/pydantic/monty/v1.3.0/docs/security.md) defines its budget as requested allocator bytes, not process resident memory (RSS). Other allocation routes and host callback allocations stay outside that count and need separate accounting.

## Stop a run before replacing it

Stop the old execution before starting the next request. In these cases, browser and native loops reach outer timeouts. Monty's required loop result is `MontyRuntimeError` wrapping built-in `TimeoutError` at `feed_run`. An outer timeout wouldn't establish that the feed limit fired.

Browser termination [aborts execution and discards queued tasks](https://html.spec.whatwg.org/multipage/workers.html#terminate-a-worker). The Linux supervisor cleans its owned process group for these benign children. Processes that change group or session fall outside that cleanup. Neither mechanism promises hard real-time scheduling or adversarial containment.

A fresh worker, child or Monty session runs the calculation successfully after each loop. That demonstrates replacement, not interrupted-state survival, rollback or undoing completed host effects. Discard a Monty session after a time or memory limit, even in a pool.

## Versions and test coverage

The selected versions are native CPython 3.13.14, Pyodide 314.0.7 reporting Python 3.14.2, and [pydantic-monty](https://pypi.org/project/pydantic-monty/1.3.0/), `pydantic-monty-client` and `pydantic-monty-runtime` at 1.3.0. The comparison covers five native cases, five Monty cases and seven browser cases.

The Linux cases were repeated with this source in an existing environment. Browser results come from the earlier run of the unchanged browser code, including the corrected validator and ten separate shape controls. Installation from scratch hasn't been tested.

Before choosing a runtime, run the most demanding package operation with data supplied through the intended interface. Check that the caller accepts the reply. If later calculations depend on earlier state, decide how the application will rebuild that state after the worker or session ends.
