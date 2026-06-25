# hget — A Minimal HTTP/1.0 Client Built from Raw Sockets

A command-line HTTP client written in Python using only raw sockets and the
standard library. No `urllib`, no `requests`, no `socket.gethostbyname()` —
every layer of the stack is implemented explicitly to make the underlying
protocols visible.

Built as Lab 0 for the Networks and Distributed Systems course at FaMAF (UNC).

---

## Overview

`hget` downloads a URL and saves the body to a file. What makes it different
from a one-liner with `urllib` is that it does everything from first principles:

- **DNS**: sends a hand-crafted UDP query to Quad9 (`9.9.9.9:53`) and parses
  the binary response according to RFC 1035.
- **TCP**: opens a raw `SOCK_STREAM` socket and connects to the resolved IP.
- **HTTP/1.0**: formats the request line and headers manually, then reads the
  response until the server closes the connection.

The goal is pedagogical: by not hiding behind high-level abstractions, the
implementation makes the role of each protocol layer explicit.

---

## Features

- Custom DNS resolver over UDP (RFC 1035) — no `socket.gethostbyname()`
- Raw TCP client speaking HTTP/1.0 directly
- Port extraction from URL (e.g. `http://host:8080/path`)
- Verbose diagnostic output to stderr: hostname, resolved IP, request sent,
  status line, and response headers
- Output file configurable via `-o`
- Zero runtime dependencies — `hget.py` uses only the Python standard library

---

## How It Works

### DNS Resolution (`dns_resolve`)

1. A random 16-bit query ID is generated and a type-A DNS query is built with
   `struct.pack` according to the RFC 1035 wire format: a 12-byte header
   (ID, flags `0x0100` for recursion desired, and four count fields), followed
   by the encoded QNAME and `QTYPE=1 / QCLASS=1`.

2. The QNAME is encoded by `_dns_encode_name`: each label is prefixed with its
   byte length and the sequence is terminated with a zero byte
   (`\x03foo\x03bar\x00` for `foo.bar`).

3. The query is sent over a UDP socket to `9.9.9.9:53` with a 5-second timeout.

4. The binary response is parsed by `_dns_parse_response`: it validates the
   response ID and QR/RCODE flags, then iterates over the answer section.
   `_dns_skip_name` handles DNS compression pointers (the `0xC0` two-byte
   pointer form). Each resource record is inspected by `_dns_parse_one_rr`;
   the first type-A record (`RTYPE=1`, `RDLENGTH=4`) is returned as a dotted
   IPv4 string.

`localhost` is resolved to `127.0.0.1` immediately, without a network query.

### HTTP Request/Response (`connect_to_server`, `send_request`, `get_response`)

1. A `SOCK_STREAM` TCP socket is opened and connected to the resolved IP and
   port.

2. The request is sent as a single `bytes` object:
   ```
   GET <full-url> HTTP/1.0\r\n
   Host: <hostname>\r\n
   \r\n
   ```
   The `Host` header is always included, even though HTTP/1.0 does not require
   it, for compatibility with name-based virtual hosting.

3. The response is read line by line until the blank line that separates headers
   from the body. The status line is checked for HTTP code `200`. After the
   header section, the body is read in 4 096-byte chunks and written to the
   output file.

4. End-of-body detection relies on the HTTP/1.0 connection model: the server
   signals the end of the response by closing the TCP connection.
   `connection.recv()` returns `b''` when the peer has closed its side, so
   the read loop `while data != b'': ...` terminates naturally — no
   `Content-Length` parsing or chunked transfer decoding is needed.

---

## Usage

```bash
python3 hget.py http://example.com/
python3 hget.py -o page.html http://www.famaf.unc.edu.ar/
python3 hget.py http://localhost:8080/
```

Diagnostic output is written to stderr; only the downloaded body goes to the
output file.

A minimal local HTTP/1.0 server for manual testing is included:

```bash
python3 servidor_lab0.py 8080
# in another terminal:
python3 hget.py http://localhost:8080/
```

---

## Testing & Code Quality

Tests are split across two files:

- **`hget-test.py`** — unit tests using a `FakeSocket` stub (no network
  required) covering request formatting, line reading, response parsing, DNS
  helpers, and URL parsing. Includes one `@pytest.mark.integration` test that
  spawns `servidor_lab0.py` in a subprocess and performs a full
  download cycle.

- **`test_metrics.py`** — quality gate tests:
  - **Cyclomatic complexity**: no function in `hget.py` may exceed a McCabe
    complexity of **8** (checked with `radon`).
  - **Static analysis**: `ruff check` must report zero errors or warnings.
  - **Coverage**: enforced by `pytest-cov`; the minimum threshold is **65%**
    (configured in `pyproject.toml` via `--cov-fail-under`).

Run the full suite:

```bash
pytest -v
```

Run the self-grading script (mirrors the criteria used for submission):

```bash
python3 grade.py
```

---

## Build & Run

```bash
# 1. Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 2. Install test/quality dependencies
pip install -r requirements.txt

# 3. Run
python3 hget.py http://example.com/
```

`hget.py` itself has no dependencies beyond the Python standard library and
works with Python 3.9+.

---

## Course Context

**Networks and Distributed Systems — Lab 0**
FaMAF, Universidad Nacional de Córdoba (UNC)

Individual assignment. I implemented the DNS resolver, HTTP client, and test
suite from scratch as a first contact with low-level socket programming in
Python.
