---
title: "Python runtimes for agent tools"
date: 2026-10-27T00:00:00+04:00
draft: true
author: ["Tejesh Vaish"]
tags: ["python", "webassembly"]
showToc: true
comments: false
---

Suppose you're building an analyst assistant in the browser and want to give it a Python tool. Start with `values = [2, 3, 5]`. Can the tool run this expression from `lab/task.py` and send back the count and sum?

```python
{
    "count": len(values),
    "sum": sum(values)
}
```

Pyodide returned `{"count": 3, "sum": 10}`. A separate test loaded NumPy 2.4.6 and got 10 from the same values. Both calculations ran in the browser because Pyodide runs CPython compiled for WebAssembly.

Native CPython and Monty got the same answer from the same expression. The native case runs in a child process on the host. Monty is an interpreter, and the app embedding it decides which operations to expose.

The test script uses fixed inputs and runs on its own, without an assistant or language model. The NumPy case is specific to Pyodide. You can find the setup and exact settings in the [complete source](https://github.com/tridev-tej/technical-blog-examples/tree/686143f355958964e6ca9eaf4342823abf7f50f6/pyodide) and [README](https://github.com/tridev-tej/technical-blog-examples/blob/686143f355958964e6ca9eaf4342823abf7f50f6/pyodide/README.md).

For a browser-based assistant, I'd start with Pyodide if the packages I need can work with the data the app passes in. If the tool needs host-native libraries or operating system (OS) integration, I'd look at native CPython. I'd consider Monty if it supports the Python I need and the app can handle any work that needs to happen on the host.

## Test the package operation

Pyodide's [FAQ](https://pyodide.org/en/stable/usage/faq.html) distinguishes pure-Python wheels from C, Fortran and Rust extensions that need compatible Pyodide builds. A host-native wheel can't replace a browser build. The [Rust/PyO3 post](https://blog.pyodide.org/posts/rust-pyo3-support-in-pyodide/) explains the target and linking constraints; the [314 release](https://blog.pyodide.org/posts/314-release/) describes the PyEmscripten package platform.

Once the import works, try the operation you actually need. If your code starts a thread or launches another process, test that call. The FAQ lists restrictions on threads, multiprocessing and subprocesses, and a successful import or system information query tells you little about whether those calls work. Native CPython uses the host's package ecosystem, so the interpreter, OS and binaries still have to match. Host-native dependencies and launching executables were outside the test.

Monty can share state across code submissions, called feeds, within a session. The application embedding Monty is the host; code running inside the interpreter is the guest. Check that the calculation fits Monty's supported Python. Its [1.3.0 security document](https://raw.githubusercontent.com/pydantic/monty/v1.3.0/docs/security.md) specifies no guest foreign function interface (FFI) or native-code dependencies. A callback, or host function exposed to the guest, can use a host package. An exposed host module can too. That work runs on the host and doesn't add native-extension support to the guest.

## What can each runtime read?

The file test uses a small marker with known text, so the caller can check the reply. Native CPython reads the file from its path on the host. The child inherits its parent's permissions and runs without an OS filesystem sandbox. Despite the name, `marker_grant` performs the same read with the same permissions as `marker_default`.

What happens if you pass the same path to Pyodide in the browser? This is what the worker sent back.

```json
{
  "status": "result",
  "value": "FileNotFoundError"
}
```

The marker is missing from the browser filesystem. The adapter catches the exception and returns the string `"FileNotFoundError"`, which the test accepts as the expected result. This case only checks a browser file lookup. Showing that a host read was denied would require a different test.

Monty returns an error because the guest has no read operation available.

```json
{
  "status": "error",
  "phase": "feed_run",
  "wrapper_type": "MontyRuntimeError",
  "error_type": "PermissionError"
}
```

Both replies show selected fields. At `feed_run`, Monty wraps the built-in `PermissionError` in a `MontyRuntimeError`. The pathname gives each runtime only a name. To make the reads work, give browser Pyodide the file bytes or expose a host read operation to Monty.

![Interfaces used by the three lab environments](/images/python-execution-authority.svg)

The diagram shows the interfaces used in this lab. These tests don't establish general sandbox guarantees. Exposed JavaScript and host callbacks add the access described here.

If your Python code expects a file, copy the marker into Pyodide's default in-memory [MEMFS filesystem](https://pyodide.org/en/stable/usage/file-system.html). You can also pass the text as a JavaScript property and read it from Python. Here are the two worker functions. The surrounding module initializes Pyodide.

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

The MEMFS copy lives in memory and lasts only as long as that filesystem does. The host directory stays outside it.

The JavaScript route gives Python access to the worker's existing JavaScript environment. When you import from `js`, Pyodide looks in `globalThis`, as the [314.0.7 interop guide](https://pyodide.org/en/314.0.7/usage/type-conversions.html) explains. Adding `labMarker` leaves other globals reachable. A custom Python globals dictionary only changes the Python namespace. The bridge test covers one property read. The page DOM, credentials, network services and host files are outside its scope.

In the next Monty case, the guest gets a zero-argument `read_marker()` callback. The host picks the path and reads the file with its own process permissions, outside the interpreter. If the callback accepted a path from the guest, the guest could choose which file the host reads. The earlier denial came from Monty's rules about exposed operations. OS sandboxing of the worker would need its own test.

The [mcp-run-python maintainers](https://raw.githubusercontent.com/pydantic/mcp-run-python/main/README.md) cite safety and latency concerns in retiring their tool. [langchain-sandbox](https://raw.githubusercontent.com/langchain-ai/langchain-sandbox/main/README.md) separately says it is unmaintained and advises against production use. Each notice concerns its own tool; neither establishes a universal verdict on Pyodide.

## Check replies before updating the application

Native CPython sends JSON over pipes, Pyodide sends worker messages, and Monty returns a feed result after any host calls. Whichever runtime you use, make sure the reply belongs to the active request. Both validators require a matching run ID, exact keys in the reply and its value, and an exact count of 3 and sum of 10. Using that fixed answer gives the caller an independent check. Running the task expression again to get the expected answer could repeat the same mistake.

In JavaScript, an array can have named `count` and `sum` properties, and those properties can survive `structuredClone`. That's why the browser validator rejects arrays both as the reply envelope and as its value. The Python validator for native CPython and Monty requires exact dictionaries and integer types.

The browser reader removes its listeners after one reply or a timeout, so a late answer can't satisfy a later call. These checks establish the accepted result for this fixed test; they don't prove arbitrary model answers correct.

## Give the whole request a deadline

I'd start the clock as soon as the app starts gathering inputs and keep it running until the app accepts a reply. That gives you a deadline for setup and host calls too.

In the browser test, fetching the data and reading its JSON body share one abort deadline. The worker gets a separate budget to start and load Pyodide. Then the case clock starts, which also covers loading NumPy. You can see the timers in `lab/main.mjs`. The [worker guide](https://pyodide.org/en/stable/usage/webworker.html) walks through initialization and messages.

In the Linux test, the supervisor starts the child, writes to its input and closes it. Only then does the readiness clock start. The output clock starts once the child is ready. You need a deadline that also covers process creation and input handling. The gap is visible in the source. A startup hang wasn't reproduced. The [subprocess documentation](https://docs.python.org/3.13/library/subprocess.html#subprocess.Popen.communicate) also distinguishes creation from timed communication.

Monty's feed clock leaves out the time spent compiling the submitted code and pauses while it waits for the host. Each feed gets a fresh allowance, so set a deadline for the whole session too. Decide when to time out host operations and cancel outstanding host work when the session ends. The marker callback only reads a local file. A speed comparison would need equivalent timing, since the runtimes measure different parts of the request.

## Budget memory separately

The lab leaves Monty's `max_memory` unset. Browser Pyodide and native CPython run without a per-run memory cap, so memory limits are outside the test.

If the application needs a memory bound, account for allocations outside the guest limit. Monty's [1.3.0 security guidance](https://raw.githubusercontent.com/pydantic/monty/v1.3.0/docs/security.md) defines its budget as requested allocator bytes, not process resident memory (RSS). Other allocation routes and host callback allocations stay outside that count and need separate accounting.

## Stop a run before replacing it

What happens if the Python code gets stuck in a loop? In the browser and native tests, the caller's timeout fires. The Monty test requires a `MontyRuntimeError` wrapping the built-in `TimeoutError` at `feed_run`. That specific error tells you Monty's own feed limit fired. In each case, stop the old execution before starting the next request.

Browser termination [aborts execution and discards queued tasks](https://html.spec.whatwg.org/multipage/workers.html#terminate-a-worker). The Linux supervisor cleans its owned process group for these benign children. Processes that change group or session fall outside that cleanup. Neither mechanism promises hard real-time scheduling or adversarial containment.

After each loop, the original calculation worked again in a fresh worker, child process or Monty session. The test covers restarting with fresh state. Recovery of interrupted state and rollback of completed work were outside its scope. If a Monty session hits a time or memory limit, discard it even if you're using a pool.

## Versions and test coverage

The selected versions are native CPython 3.13.14, Pyodide 314.0.7 reporting Python 3.14.2, and [pydantic-monty](https://pypi.org/project/pydantic-monty/1.3.0/), `pydantic-monty-client` and `pydantic-monty-runtime` at 1.3.0. The comparison covers five native cases, five Monty cases and seven browser cases.

The Linux cases ran again with this source in an existing environment. The browser results are from the earlier run of the same browser code, including the corrected validator and ten separate checks of reply shape. A fresh install is still untested.

If you're trying this in your own app, start with the hardest package operation you need. Give it data the way your app would, then check that the app accepts the reply. If your next calculation relies on state from this one, work out how you'll rebuild that state when a worker or session ends.
