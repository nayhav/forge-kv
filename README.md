# Go Redis Clone

A from-scratch, in-memory key-value store written in Go that speaks the Redis wire protocol over plain TCP. It's not trying to replace Redis — it's a learning project that reimplements the pieces that make Redis tick: expiring keys, atomic transactions, pub/sub messaging, and LRU-based eviction.

## What's Inside

### The Basics
- `GET key`
- `SET key value`
- `DEL key`
- `INCR key` / `DECR key`
- `FLUSHALL`
- `PING`
- `QUIT`

### Key Expiration
- `EXPIRE key seconds` sets a TTL on a key
- `TTL key` tells you how much time a key has left
- Expired keys get swept up automatically in the background — no need to wait for a read to trigger cleanup

### Transactions
- `MULTI` opens a transaction block
- `EXEC` runs everything that's been queued
- `DISCARD` throws the queued commands away instead
- Each connection gets its own isolated queue, and commands run atomically once executed

### Pub/Sub
- `SUBSCRIBE channel`
- `PUBLISH channel message`
- Supports multiple channels at once for real-time message delivery between clients

### LRU Eviction
- Once memory usage crosses a configured threshold, the least-recently-used keys get evicted automatically
- Built on top of Go's `container/list` package to keep updates at O(1)
- Wired directly into `GET`, `SET`, and `DEL` so recency tracking stays accurate

### RESP Protocol
- Implements the RESP wire format (`*`, `$`, `+`, `-`, `:`) so it plays nicely with the standard `redis-cli` and other existing Redis tooling

### The Server Itself
- Binds to `tcp://0.0.0.0:6380`
- Handles multiple client connections at the same time
- Doesn't fall over on bad input — malformed commands are handled gracefully

## Layout of the Repo

```
redis-clone/
├── cmd/                         # Entry points for the binaries
│   ├── bench/
│   │   └── bench.go             # Benchmarking client
│   └── server/
│       └── main.go              # Boots the server
│
├── internal/                    # Where the actual logic lives
│   ├── cache/                   # In-memory storage
│   │   ├── lru.go               # LRU eviction logic
│   │   ├── lru_test.go
│   │   ├── store.go             # Key-value store + expiration
│   │   └── store_test.go
│
│   ├── command/                 # Parses and runs commands
│   │   ├── handler.go           # GET, SET, DEL, etc.
│   │   └── handler_test.go
│
│   ├── persistence/              # RDB/AOF persistence
│   │   ├── rdb.go
│   │   └── rdb_test.go
│
│   ├── protocol/                 # RESP protocol implementation
│   │   ├── buffer_writer.go     # Buffered output writer
│   │   ├── parser.go            # RESP3-compatible parser
│   │   └── parser_test.go
│
│   ├── pubsub/                   # Pub/sub broker
│   │   ├── pubsub.go
│   │   └── pubsub_test.go
│
│   ├── session/                  # Per-connection session state
│   │   ├── session.go
│   │   └── session_test.go
│
│   └── transaction/               # MULTI/EXEC handling
│       └── transaction.go
│
├── img/                          # Diagrams
│   ├── sys.png
│   └── sys2.png
│
├── dump.rdb                      # Sample RDB file for testing persistence
├── redis-clone                   # Compiled binary (via Makefile)
├── go.mod
├── LICENSE
├── Makefile
└── README.md
```

## Getting It Running

Start the server:

```bash
go run cmd/server/main.go
```

Then connect to it like you would any Redis instance:

```bash
redis-cli -p 6380
```

To run the benchmark client:

```bash
go run cmd/benchmark/main.go -clients=50 -requests=100
```

## Running the Tests

> **Heads up:** the test suite is still a work in progress. Coverage isn't complete everywhere yet, and a few known issues are still being sorted out.

Each package has its own test target:

```bash
go test ./internal/cache -v        # cache package
go test ./internal/command -v      # command handling: GET, SET, SET EX, DEL, etc.
go test ./internal/protocol -v     # RESP parsing
go test ./internal/session -v      # session management
go test ./internal/persistence -v  # persistence
go test ./internal/pubsub -v       # pub/sub
```

## Inspiration

This wasn't built in a vacuum — a handful of resources shaped how it came together, adapted along the way to fit my own preferences:

- [redis-internals](https://github.com/) — internals reference
- *Writing a Redis clone in Go from scratch* (blog post)
- *Go, for Distributed Systems* by Russ Cox
- *Designing Data-Intensive Applications* by Martin Kleppmann
- And, unavoidably these days, some help from LLMs (GPT-4o, Claude, Gemini)

## Want to Contribute?

1. Fork the repo
2. Cut a new branch:
   ```bash
   git checkout -b feature/my-update
   ```
3. Make your changes, then commit and push:
   ```bash
   git add .
   git commit -m "Added my update"
   git push origin feature/my-update
   ```
4. Add tests if it makes sense for what you changed
5. Run the test suite locally to make sure nothing's broken
6. Open a pull request and explain what you did and why

## Roadmap

- [ ] Get the unit tests fully passing
- [ ] Flesh out proper LRU caching support
- [ ] Add real persistence (RDB/AOF)
- [ ] Build a standalone CLI client
- [ ] Improve the benchmarking tool
