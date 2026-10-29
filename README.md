# ringasgi

**An experimental io_uring-based HTTP server for Python ASGI apps (FastAPI, Starlette).**

> **Status: idea / early research.** Nothing is built yet. This repository records the
> idea and tracks the work as it happens.
>
> **Idea first written down:** 29 October 2026
> **Author:** Shubham Pawar ([@Shubhtistic](https://github.com/Shubhtistic))

---

## The Idea

Python ASGI servers (uvicorn, hypercorn, Granian) handle network I/O through `epoll`.
Linux `io_uring` is a newer kernel interface that batches I/O operations through shared
ring buffers, which can cut system call overhead sharply.

Servers written in Rust, C and Zig already use it, and there are a few experimental
Python ones. I have not found one that serves **ASGI applications such as FastAPI** on
top of io_uring.

**ringasgi is an attempt to build exactly that:**
a Rust core using io_uring, embedded in Python, that runs any ASGI app.

```
 client ──TCP──▶  ringasgi (Rust, io_uring, one ring per thread)
                      │  parse HTTP/1.1
                      │  build ASGI scope
                      ▼
                 Python ASGI app  (FastAPI / Starlette / any ASGI callable)
                      │
                      ▼
                  response ──▶ io_uring write ──▶ client
```

## Why this might not work (honest version)

A typical FastAPI request spends most of its time in Python, Pydantic and the database,
not in the kernel network stack. An io_uring server can only improve the part that
`epoll` was already handling reasonably well.

```
Where one request's time goes
  kernel network + syscalls      ####            <- io_uring helps here
  HTTP parsing                   ##              <- Rust servers already fast
  ASGI glue (GIL-bound)          ######
  framework + validation         ########
  application code (DB, etc.)    ################
```

So the **first question of this project is not "can I beat Granian?"** but
**"does io_uring help at all once Python is involved?"** Both answers will be published,
including a negative result.

## Goals

- Learn io_uring properly by building something real on it
- Run an unmodified ASGI app (a bare ASGI function, then Starlette, then FastAPI)
- Publish honest, reproducible benchmarks against uvicorn and Granian
- Document what worked, what didn't, and why

## Non-goals (for now)

- Beating Granian in every scenario
- HTTP/2, HTTP/3, WebSockets (maybe later)
- Production use

## Roadmap

- [ ] **Stage 0: Learn.** An io_uring echo server (liburing or Rust `io-uring`/`monoio`)
- [ ] **Stage 1: Premise check.** Rust HTTP/1.1 server on io_uring vs the same server on epoll, no Python. Publish results. If the gain is negligible, document it and reassess
- [ ] **Stage 2: Python embedding.** PyO3, ASGI scope/receive/send, GIL handling
- [ ] **Stage 3: Benchmarks.** Bare ASGI, Starlette, FastAPI vs uvicorn and Granian, with flamegraphs
- [ ] **Stage 4: Correctness.** Keep-alive, streaming and chunked bodies, ASGI lifespan, graceful shutdown
- [ ] **Stage 5: Hardening.** Timeouts, limits, TLS, fuzzing

## Architecture (planned)

```
   +-----------------------------------------------+
   |  process                                      |
   |   thread 0: ring 0 ─ accept/read/write ─┐     |
   |   thread 1: ring 1 ─ accept/read/write ─┼──▶ Python (GIL)
   |   thread N: ring N ─ accept/read/write ─┘     |
   |   (SO_REUSEPORT, one ring per thread)         |
   +-----------------------------------------------+
```

Open design questions (to be answered with experiments, not guesses):
- How cheaply can the Rust side hand a request to Python and get a response back?
- Per-thread rings vs a shared ring?
- Multishot accept and receive, registered buffers: how much do they matter?

## Benchmark methodology (committed in advance)

- Same machine, same kernel, same app code for every server
- Three app tiers: bare ASGI function, Starlette route, FastAPI route with a Pydantic model
- Report: requests/sec, p50/p95/p99 latency, CPU use, memory, at several concurrency levels
- Publish scripts, raw data and kernel/hardware details so results can be reproduced
- Negative and neutral results get published too

## Requirements (expected)

- Linux, kernel 6.x (exact minimum will be set by the features used)
- Python 3.12+
- Rust toolchain
- A runtime that allows io_uring (some container runtimes and sandboxes restrict it)

## Related work

This project builds on ideas from, and will be compared against:

- [Granian](https://github.com/emmett-framework/granian): Rust-based ASGI/WSGI/RSGI server
- [uvicorn](https://github.com/encode/uvicorn) and [hypercorn](https://github.com/pgjones/hypercorn): standard Python ASGI servers
- [tokio-uring](https://github.com/tokio-rs/tokio-uring), [monoio](https://github.com/bytedance/monoio), [glommio](https://github.com/DataDog/glommio): Rust io_uring runtimes
- [one-ring-http](https://pypi.org/project/one-ring-http/): Python HTTP/1.1 server on io_uring (not ASGI)

If you know of an existing io_uring ASGI server, please open an issue. Pointing to prior
art is welcome.

## Contributing

The project is at the idea stage. Discussion in issues is welcome. Code contributions will
open up once Stage 1 gives a go/no-go answer.

## License

MIT
