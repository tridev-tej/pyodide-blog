---
title: "Choosing a Python runtime for a browser assistant"
date: 2026-10-27T00:00:00+04:00
draft: true
author: ["Tejesh Vaish"]
tags: ["python", "webassembly"]
showToc: true
comments: false
---

If you're building an analyst assistant in the browser, choose its Python runtime around the operation you need. Can the required packages work with data supplied by the browser, or does the operation need host libraries and operating system access? The answer helps you choose where the tool runs and what access the application gives it.

The [comparison lab](https://github.com/tridev-tej/technical-blog-examples/tree/686143f355958964e6ca9eaf4342823abf7f50f6/pyodide) runs the same calculation in browser Pyodide, native CPython in a child process, and Monty. Fixed code keeps language-model behavior out of the comparison so you can inspect how the application supplies data and handles results. The cases don't establish a performance ranking or containment of arbitrary code.

The starting calculation in `lab/task.py` uses `values = [2, 3, 5]`:

```python
{
    "count": len(values),
    "sum": sum(values)
}
```

All three runtimes returned `{"count": 3, "sum": 10}`. A separate browser case loaded NumPy 2.4.6 and returned 10 for the sum of the same values. The lab also tries a file read and a stuck loop, then checks that the calculation works in a fresh runtime.

## Run the comparison

Use a fresh directory on Linux x86_64 with Git, CPython 3.13.14, pip, curl, and `sha256sum`. Keep Python assertions enabled. Browser cases need module workers. The [README](https://github.com/tridev-tej/technical-blog-examples/blob/686143f355958964e6ca9eaf4342823abf7f50f6/pyodide/README.md) documents the setup.

Clone and pin the source, then run these commands in order. Stop if a command fails and fix its cause. Before setup, check that `python3.13 --version` prints `Python 3.13.14`.

```sh
git clone https://github.com/tridev-tej/technical-blog-examples.git
cd technical-blog-examples
git checkout --detach 686143f355958964e6ca9eaf4342823abf7f50f6
cd pyodide
sha256sum -c MANIFEST.sha256
python3.13 --version
sh commands/01-setup.sh
cd lab
sh ../commands/08-run-native.sh
sh ../commands/09-stage-assets.sh
```

Setup creates `.venv` beside `lab`; its internal `cd` doesn't change your shell's directory, so keep the explicit `cd lab`. The `08-run-native.sh` script prepares `fixture.json` and `marker.txt`, then runs five native cases and five Monty cases. Each sequence runs the calculation, a default file read, a supplied read, a loop, and the calculation again in fresh state. Both calculation results must have count 3 and sum 10.

Check that loopback port 43126 is free, then serve from the same `lab` directory:

```sh
sh ../commands/13-serve.sh
```

Open `http://127.0.0.1:43126/` and click **Run fixed cases**. Retain the seven JSON rows and any browser errors. These cases add a JavaScript property read and NumPy to the five-case sequence. Check that `marker.txt` still matches the marker in `fixture.json`. The server exposes only `lab/`; stop it with Ctrl-C.

Some cases expect a file error or timeout. The checks require the specific outcomes below; an unrelated failure doesn't count as success.

The selected versions are native CPython 3.13.14, Pyodide 314.0.7 reporting Python 3.14.2, and [pydantic-monty](https://pypi.org/project/pydantic-monty/1.3.0/), `pydantic-monty-client`, and `pydantic-monty-runtime` at 1.3.0. For this revision, all ten native and Monty cases passed in a new Linux virtual environment using an existing CPython installation. All seven browser cases and ten reply-shape checks passed in Chrome 149. The README's fresh-install caveat predates this check. These results cover the fixed cases; your package operation still needs its own check.

## Check the operation your package needs

Pyodide runs CPython compiled for WebAssembly. Its [FAQ](https://pyodide.org/en/stable/usage/faq.html) distinguishes pure-Python wheels from C, Fortran, and Rust extensions that need compatible Pyodide builds. A host-native wheel can't replace a browser build. The [Rust/PyO3 post](https://blog.pyodide.org/posts/rust-pyo3-support-in-pyodide/) explains the target and linking constraints; the [314 release](https://blog.pyodide.org/posts/314-release/) describes the PyEmscripten package platform.

Once an import works, exercise the operation your tool needs. If it starts a thread or launches a subprocess, test that call against the restrictions in the FAQ. An import or system information query doesn't establish that those operations work. NumPy was tested only in the browser, with one calculation; package parity is outside this comparison.

Native CPython uses the host's package ecosystem, so its interpreter, operating system, and binaries still need to match. I'd consider it for a tool that needs a host-native library or executable. Those dependencies and executable launches were outside this test.

Monty is an interpreter. A code submission is called a feed, and a session can retain state across feeds. Check that your calculation fits its supported Python. The embedding application is the host; Python inside Monty is the guest. Its [1.3.0 security document](https://raw.githubusercontent.com/pydantic/monty/v1.3.0/docs/security.md) specifies no guest foreign function interface (FFI) or native-code dependencies. A callback or exposed host module can use a host package. That work runs on the host and doesn't add native-extension support to the guest.

## Give the tool the data it needs

The file cases use a marker with known text. Native CPython reads it from the host path with its parent's ordinary permissions, without an operating system filesystem sandbox. The two scenarios, `marker_default` and `marker_grant`, perform the same host read. The second name doesn't mean that the application granted new permissions.

Passing the host pathname to browser Pyodide produces this reply, with fields omitted for brevity:

```json
{
  "status": "result",
  "value": "FileNotFoundError"
}
```

The file is missing from the browser filesystem. The adapter catches `FileNotFoundError` and returns its name as a successful result. This checks a missing-file lookup; it doesn't demonstrate denial of a host read.

Monty's default read has no exposed read operation. At `feed_run`, it returns a `MontyRuntimeError` wrapping built-in `PermissionError`. The next case supplies a zero-argument `read_marker()` callback. The host chooses the file and reads it with its own process permissions. A callback accepting a path from the guest would let the guest choose which file the host reads. Keep path selection with the host when the tool only needs one file.

![Interfaces used by the three lab environments](/images/python-execution-authority.svg)

For browser Python that expects a file, copy the bytes into Pyodide's default in-memory [MEMFS filesystem](https://pyodide.org/en/stable/usage/file-system.html). If Python needs only text, the application can supply a JavaScript property. These functions come from `lab/worker.mjs`; the surrounding module initializes Pyodide:

```javascript
function markerGrant(message) {
  pyodide.FS.writeFile("/marker.txt", message.marker);
  return pyodide.runPython("open('/marker.txt').read()");
}

function bridge(message) {
  globalThis.labMarker = message.marker;
  return pyodide.runPython("from js import labMarker\nstr(labMarker)");
}
```

The MEMFS copy lasts only as long as that filesystem does and doesn't mount the host directory. Imports from `js` use the worker's existing `globalThis`, as the [314.0.7 interop guide](https://pyodide.org/en/314.0.7/usage/type-conversions.html) explains. Adding `labMarker` leaves other worker globals reachable, so review what that environment exposes. A custom Python globals dictionary changes the Python namespace without revoking that JavaScript access.

The bridge case checks one property read. The page DOM, credentials, network services, and host files are outside its scope. The diagram describes these interfaces, with no general sandbox guarantee. An operating system sandbox around the worker would need separate verification.

The [mcp-run-python maintainers](https://raw.githubusercontent.com/pydantic/mcp-run-python/main/README.md) cite safety and latency concerns in retiring their tool. [langchain-sandbox](https://raw.githubusercontent.com/langchain-ai/langchain-sandbox/main/README.md) separately says it is unmaintained and advises against production use. Each notice concerns its own tool. When using Pyodide, inspect what the embedding JavaScript environment can do and which interfaces Python can reach.

## Accept only the expected reply

Your application still has to decide whether a result belongs to the active request and can update its state. Native CPython sends JSON over pipes, Pyodide sends worker messages, and Monty returns a feed result after any host calls. The browser's existing `lab/contract.mjs` validates the fixed calculation:

```javascript
function exactKeys(value, keys) {
  if (value === null || typeof value !== "object" || Array.isArray(value)) return false;
  return Object.keys(value).sort().join() === keys;
}

export function acceptFixture(reply, runId, active) {
  if (!active || !exactKeys(reply, "run_id,status,value")) return false;
  if (reply.run_id !== runId || reply.status !== "result") return false;
  const value = reply.value;
  if (!exactKeys(value, "count,sum")) return false;
  return value.count === 3 && value.sum === 10;
}
```

The validator requires an active request, matching run ID, and exact keys in both the envelope and value. Arrays can have named `count` and `sum` properties that survive `structuredClone`, so it rejects arrays at both levels. For the same calculation, `lab/contract.py` requires exact dictionaries and integer types in native CPython and Monty.

Count 3 and sum 10 form the independent expected answer, or oracle. Keep that expectation separate from the expression under test; evaluating the same expression again could repeat the same mistake. This checks one known answer. Generated answers need checks suited to the task.

The browser reader in `lab/main.mjs` removes its listeners after a reply or timeout so a late answer can't satisfy a later call. Preserve that lifecycle handling when adapting the result checks.

## Bound the request and replace stopped runs

I'd start an application deadline when the app begins gathering inputs and keep it running until the app accepts the reply. The lab uses separate clocks, so account for the gaps when adapting it.

In `lab/main.mjs`, fetching the fixture and reading its JSON body share one abort deadline. Worker startup and Pyodide loading get a separate budget, followed by the case clock, which also covers NumPy loading. The [worker guide](https://pyodide.org/en/stable/usage/webworker.html) explains initialization and messaging.

In `lab/run_native.py`, process creation and input write/close happen before the readiness clock starts. Output gets its own clock after the child is ready. Include those earlier steps in your application's deadline; a startup hang wasn't reproduced here. The [subprocess documentation](https://docs.python.org/3.13/library/subprocess.html#subprocess.Popen.communicate) also distinguishes process creation from timed communication.

Monty's feed clock excludes compilation of the submitted source and pauses during host calls. Each feed gets a fresh allowance, so give host work and the whole session deadlines too. Cancel outstanding host work when ending the session. The marker callback only reads a local file; it doesn't exercise slow host services. A speed comparison would need equivalent timing across the runtimes.

Memory needs its own policy. This lab leaves Monty's `max_memory` unset and gives browser Pyodide and native CPython no per-run memory cap. Monty's [memory budget](https://raw.githubusercontent.com/pydantic/monty/v1.3.0/docs/security.md) counts requested allocator bytes, not process resident memory (RSS). Other allocation routes and host callback allocations need separate accounting.

The loop case checks whether execution can end before the next calculation. Browser and native runs reach the caller's outer timeout. Monty must return a `MontyRuntimeError` wrapping built-in `TimeoutError` at `feed_run`; an outer timeout doesn't satisfy that check. Stop the old execution before accepting a replacement request.

Browser termination [aborts execution and discards queued tasks](https://html.spec.whatwg.org/multipage/workers.html#terminate-a-worker). The Linux supervisor cleans its owned process group for these benign children; processes that change group or session fall outside that cleanup. Neither mechanism promises hard real-time scheduling or adversarial containment.

After each loop, the calculation worked again in a fresh worker, child process, or Monty session. Replacement starts with fresh state; recovery of interrupted state and rollback of completed work were outside the test. Monty's security guidance says to discard a session after a time or memory limit, including when using a pool.

## Adapt the lab to your tool

I'd start with Pyodide when browser-compatible packages can process the data the app supplies. Native CPython fits host-native libraries or operating system integration. Monty is worth considering when it supports the calculation and the application can provide any host operations it needs. The marker cases show what supplying data involves in each setup: copying bytes into the browser, reading with ordinary host permissions, or calling a host function.

Start with the hardest package operation you need. `lab/task.py` holds a single expression; keep that form or change the adapters to handle statements. Put package setup in the adapters: `lab/worker.mjs` for Pyodide, `lab/native_guest.py` for native CPython, and `lab/monty_guest.py` for Monty host work. Update prepared inputs in `lab/run_native.py` and Monty's fixed input in `lab/monty_guest.py`. Inspect `lab/main.mjs` and `lab/run_native.py` for lifecycle handling.

Update both contract files with an independently calculated expected answer. If the result shape changes, update the exact-key checks too. Then run the cases again with data supplied the way your app will supply it. For a tool that retains state between requests, decide how to rebuild that state after a worker or session ends.
