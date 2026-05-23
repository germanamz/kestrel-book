# Sockets From Scratch

Kestrel's in-memory layer is done; its persistence layer is done; what
remains is for a client to actually talk to it. That requires a network
server, and we are going to write one against raw Berkeley sockets — the
same C API every Unix server has used since the 1980s — wrapped in modern
C++. This chapter is a complete tour through the syscalls (`socket`,
`bind`, `listen`, `accept`, `read`, `write`, `close`), the C++ idiom (an
RAII wrapper around a file descriptor), and the *blocking* server design,
because before you build a non-blocking event loop in Chapter 15, you must
have done blocking right.

The C++ topic is **RAII over POSIX resources**. The WAL in Chapter 12
managed a file descriptor inline; here we generalize that pattern into a
`FileDescriptor` class that every later piece of code uses without
re-implementing the lifecycle. The other topic is **error handling at the
POSIX boundary**: turning `-1`-plus-`errno` into typed
`std::expected<T, NetworkError>` that the rest of Kestrel can compose with.

By the end, `src/net/` will have an RAII `FileDescriptor`, a `Listener`
that opens and accepts on a TCP port, and a `Connection` that reads bytes
from a socket and writes them back — enough to run an `echo` test, and the
foundation Chapters 15 and 16 build the event loop and the real protocol
dispatch on.

---

## The Berkeley sockets API in five calls

A TCP server has the same five-step shape in every C-derived language. You
already know it; the C-API names are:

```c
int s = socket(AF_INET, SOCK_STREAM, 0);     // 1. create a socket file descriptor
bind(s, &addr, sizeof(addr));                 // 2. attach it to a local IP:port
listen(s, backlog);                           // 3. mark it as a server socket
int c = accept(s, &peer_addr, &peer_len);     // 4. wait for and accept a client
read(c, buf, n) / write(c, buf, n);           // 5. exchange bytes; close when done
```

A socket *is* a file descriptor — an integer that indexes into the
kernel's per-process file table. The first four calls produce one; the
fifth uses it. `close(s)` releases it. Every socket-related bug you can
think of is one of:

- **Leaks**: a file descriptor opened and not closed. Long-running servers
  hit the per-process fd limit (`ulimit -n`) and then refuse new
  connections.
- **Use-after-close**: `read`ing or `write`ing a closed fd, which returns
  `EBADF`. Worse: if the fd has been reused for another open file, you'll
  silently read/write into the wrong stream — a real source of corruption
  in C code.
- **Double-close**: `close(fd)` then `close(fd)` again. The first call may
  succeed and free the slot; before your second call, the kernel may
  reissue the same number to a new file. The second `close` then shuts
  down something it shouldn't.

These three are the same hazard class as the memory bugs of Chapter 3 —
managing a resource by hand and getting the lifetime wrong. The C++ answer
is, again, RAII.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

Node's `net.createServer` and Python's `socket.socket` wrap these same five
calls. In high-level languages you usually never see the fd directly; the
runtime manages it through the lifetime of a JS object or a Python `with`
block. C++ doesn't have a runtime, so the wrapper is yours to write. The
silver lining: once you write it, every later piece of socket code in
Kestrel — listeners, connections, the event loop — is exception-safe and
fd-leak-safe by construction.

</div>

---

## An RAII wrapper around a file descriptor

The pattern is by now familiar: a class that owns the resource in a
private member, with explicit move and a destructor that releases it.

```cpp
// src/net/file_descriptor.hpp
#pragma once

#include <unistd.h>     // ::close

namespace kestrel::net {

class FileDescriptor {
public:
    FileDescriptor() = default;
    explicit FileDescriptor(int fd) noexcept : fd_(fd) {}
    ~FileDescriptor() { if (fd_ >= 0) ::close(fd_); }

    FileDescriptor(const FileDescriptor&) = delete;
    FileDescriptor& operator=(const FileDescriptor&) = delete;

    FileDescriptor(FileDescriptor&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
    FileDescriptor& operator=(FileDescriptor&& o) noexcept {
        if (this == &o) return *this;
        if (fd_ >= 0) ::close(fd_);
        fd_ = o.fd_;
        o.fd_ = -1;
        return *this;
    }

    int  get() const noexcept { return fd_; }
    int  release() noexcept   { int f = fd_; fd_ = -1; return f; }
    bool valid()  const noexcept { return fd_ >= 0; }

private:
    int fd_ = -1;
};

}  // namespace kestrel::net
```

Five details are worth pinning down.

**`-1` as the empty sentinel.** POSIX file descriptors are non-negative;
`-1` is the universal "not a valid fd" value. We use it as both the
default-constructed state and the moved-from state. The destructor checks
`fd_ >= 0` before closing.

**Copy is deleted, move is provided.** A copy would mean two owners trying
to close the same fd — the double-close bug above. Move transfers ownership
and zeros the source. This is exactly `std::unique_ptr`'s shape applied to
fds; in fact, you could implement `FileDescriptor` as a
`std::unique_ptr<void, fd_closer>` and many codebases do. The hand-rolled
class is a touch friendlier syntactically.

**Destructor is implicitly `noexcept`.** As Chapter 7 noted, destructors
are `noexcept` unless you fight the language. If `close` returns an error
(rare; almost always means the fd was already invalid), there's nothing
useful to do in a destructor anyway.

**`release()` for handing ownership out.** When you need to give the fd to a
C API that wants to own it (rare), `release` returns the integer and zeros
the field. Don't confuse with `get`, which keeps ownership.

**`noexcept` everywhere it's true.** Getters, the constructor from an int
literal, move operations — none of these can throw. Marking them
`noexcept` lets the optimizer drop exception-handling tables and signals
the contract to readers.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`close` can return `-1`. The two common causes are `EBADF` (already
closed: serious bug somewhere) and `EINTR` (interrupted; Linux retains
the close, BSD doesn't — portability nightmare). Production code logs the
close failure; *retrying* close is a famously dangerous gesture because of
the BSD-vs-Linux disagreement about whether the fd was actually closed. We
just log-and-move-on; `FileDescriptor` does not check the return.

</div>

---

## Opening a listening socket

With `FileDescriptor` in hand, the server setup becomes a sequence of
fallible operations each returning `expected`. The error type follows the
Chapter 7 pattern:

```cpp
// src/net/error.hpp
#pragma once
#include <string>

namespace kestrel::net {

struct NetworkError {
    std::string what;       // human-readable
    int         errno_;     // raw errno at point of failure
};

}  // namespace kestrel::net
```

The listener type:

```cpp
// src/net/listener.hpp
#pragma once

#include <expected>
#include <cstdint>

#include "net/file_descriptor.hpp"
#include "net/error.hpp"

namespace kestrel::net {

class Listener {
public:
    static std::expected<Listener, NetworkError> bind(std::uint16_t port, int backlog = 64);

    std::expected<FileDescriptor, NetworkError> accept() const;
    int fd() const { return fd_.get(); }

private:
    explicit Listener(FileDescriptor fd) : fd_(std::move(fd)) {}
    FileDescriptor fd_;
};

}  // namespace kestrel::net
```

Implementation:

```cpp
// src/net/listener.cpp
#include "net/listener.hpp"

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <cstring>

namespace kestrel::net {

static NetworkError errno_error(const char* what) {
    return { std::string(what) + ": " + std::strerror(errno), errno };
}

std::expected<Listener, NetworkError>
Listener::bind(std::uint16_t port, int backlog) {
    FileDescriptor sock(::socket(AF_INET, SOCK_STREAM, 0));
    if (!sock.valid()) return std::unexpected(errno_error("socket"));

    // SO_REUSEADDR: let us re-bind to a port that's still in TIME_WAIT
    // after a recent close. Essential for a server that restarts.
    int yes = 1;
    if (::setsockopt(sock.get(), SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes) != 0)
        return std::unexpected(errno_error("setsockopt SO_REUSEADDR"));

    sockaddr_in addr{};
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);    // listen on all interfaces
    addr.sin_port        = htons(port);

    if (::bind(sock.get(), reinterpret_cast<sockaddr*>(&addr), sizeof addr) != 0)
        return std::unexpected(errno_error("bind"));

    if (::listen(sock.get(), backlog) != 0)
        return std::unexpected(errno_error("listen"));

    return Listener(std::move(sock));
}

std::expected<FileDescriptor, NetworkError> Listener::accept() const {
    sockaddr_in peer{};
    socklen_t   len = sizeof peer;
    int cfd = ::accept(fd_.get(), reinterpret_cast<sockaddr*>(&peer), &len);
    if (cfd < 0) return std::unexpected(errno_error("accept"));
    return FileDescriptor(cfd);
}

}  // namespace kestrel::net
```

A few line-by-line notes.

**`SO_REUSEADDR`.** Without it, a recently-closed server cannot rebind to
its port until the kernel finishes its `TIME_WAIT` (60+ seconds). For
development that is brutal. `SO_REUSEADDR` is the standard fix.
`SO_REUSEPORT` is its more aggressive sibling for multi-process servers;
we'll mention it in Chapter 18.

**`htons` and `htonl`.** "Host to network short/long" — pre-`std::byteswap`
helpers that convert from host byte order to big-endian (network byte
order). They have been there forever and are fine to use; the
`<bit>`-based `to_be32` from Chapter 12 also works.

**`reinterpret_cast<sockaddr*>(&addr)`.** This is *the* exception to "don't
`reinterpret_cast`": the POSIX sockets API was designed before C had a
clean way to express "polymorphic address structures," and casting a
specific address family (`sockaddr_in` for IPv4, `sockaddr_in6` for IPv6)
to the base `sockaddr` is the documented, correct interface. The
`<sys/socket.h>` header essentially blesses it.

<div class="callout callout-jsts">

🟦 **Coming from JS/Python**

`net.createServer().listen(6379)` in Node hides all of `socket`, `bind`,
`listen`, and the address-family casts. Doing it by hand is one of those
exercises that, after it works once, makes every later library make sense:
you can see exactly what `listen(6379)` was sugaring over. There's no
magic — just five syscalls and an integer.

</div>

---

## Reading and writing on a connection

A connection is another file descriptor — the one `accept()` returned.
For now, blocking reads and writes are what we want; Chapter 15 will turn
those non-blocking.

A minimal `Connection` wrapper that owns its fd:

```cpp
// src/net/connection.hpp
#pragma once

#include <expected>
#include <span>
#include <cstdint>

#include "net/file_descriptor.hpp"
#include "net/error.hpp"

namespace kestrel::net {

class Connection {
public:
    explicit Connection(FileDescriptor fd) : fd_(std::move(fd)) {}

    // Blocking read. On EOF, returns 0. On error, returns NetworkError.
    std::expected<std::size_t, NetworkError> read(std::span<std::uint8_t> buf);

    // Blocking write. Loops on short writes; returns when all bytes sent.
    std::expected<void, NetworkError> write_all(std::span<const std::uint8_t> buf);

    int fd() const { return fd_.get(); }

private:
    FileDescriptor fd_;
};

}  // namespace kestrel::net
```

```cpp
// src/net/connection.cpp
#include "net/connection.hpp"

#include <unistd.h>
#include <cstring>
#include <cerrno>

namespace kestrel::net {

std::expected<std::size_t, NetworkError>
Connection::read(std::span<std::uint8_t> buf) {
    for (;;) {
        ssize_t n = ::read(fd_.get(), buf.data(), buf.size());
        if (n >= 0) return static_cast<std::size_t>(n);
        if (errno == EINTR) continue;
        return std::unexpected(NetworkError{std::string("read: ") + std::strerror(errno), errno});
    }
}

std::expected<void, NetworkError>
Connection::write_all(std::span<const std::uint8_t> buf) {
    std::size_t off = 0;
    while (off < buf.size()) {
        ssize_t n = ::write(fd_.get(), buf.data() + off, buf.size() - off);
        if (n < 0) {
            if (errno == EINTR) continue;
            return std::unexpected(NetworkError{std::string("write: ") + std::strerror(errno), errno});
        }
        off += static_cast<std::size_t>(n);
    }
    return {};
}

}  // namespace kestrel::net
```

Two points the boilerplate makes clear.

**Short reads and short writes are normal**, just as they were for the WAL.
`read` returning 5 when you asked for 1024 doesn't mean an error; it means
"that's all I have for you right now." `write` returning 700 when you asked
for 1024 means "fit that much; ask me again for the rest." The connection
type loops on writes; the read returns whatever it got and lets the caller
decide what to do (the parser from Chapter 6, called from Chapter 16, will
ask again if it needs more).

**`EOF` is `read` returning 0**, not an error. The peer closed cleanly. We
propagate it as a successful `0`-byte read; the caller (a `for` loop in
the next section) treats it as "connection done" and drops out of the
loop.

<div class="callout callout-ub">

⚠️ **UB / Gotcha**

`SIGPIPE`: by default, on Linux, writing to a socket the peer has closed
causes the kernel to deliver `SIGPIPE` to your process, which terminates
it. This is *not* what a server wants. Two mitigations: install a signal
handler to ignore `SIGPIPE` once at startup (`signal(SIGPIPE, SIG_IGN)`),
or pass `MSG_NOSIGNAL` to `send()` on every write (BSD/Linux),
respectively `SO_NOSIGPIPE` (macOS). Kestrel's `main.cpp` does the
process-wide ignore; we will revisit when Chapter 15 introduces the event
loop.

</div>

---

## A first blocking server: echo

Tie the pieces together into a server that accepts one connection at a
time and echoes its bytes back. This is not Kestrel — we won't speak RESP
until Chapter 16 — but it exercises every wrapper we just wrote.

```cpp
// src/server/main_echo.cpp  (sketch — not the final kvd)
#include <csignal>
#include <cstdio>
#include <vector>

#include "net/listener.hpp"
#include "net/connection.hpp"

int main() {
    std::signal(SIGPIPE, SIG_IGN);

    auto listener = kestrel::net::Listener::bind(6379);
    if (!listener) {
        std::fprintf(stderr, "bind: %s\n", listener.error().what.c_str());
        return 1;
    }

    for (;;) {
        auto client_fd = listener->accept();
        if (!client_fd) { std::fprintf(stderr, "accept: %s\n", client_fd.error().what.c_str()); continue; }

        kestrel::net::Connection conn(std::move(*client_fd));
        std::vector<std::uint8_t> buf(4096);
        for (;;) {
            auto n = conn.read(buf);
            if (!n || *n == 0) break;                   // error or EOF
            (void)conn.write_all({buf.data(), *n});
        }
        // conn goes out of scope here; FileDescriptor destructor closes the fd.
    }
}
```

Compile it as a separate `add_executable(echo_server ...)` target, run it,
and connect with `nc localhost 6379`. Type something; see it echoed back.
This is the entire "I can speak TCP" plumbing, in modern C++.

The server is also broken in two ways you should see as features, not
bugs, because Chapter 15 fixes them:

- **It handles one client at a time.** A second `nc` connecting while the
  first is open blocks in `accept`. We will fix this with non-blocking I/O
  and a `kqueue`/`epoll`-driven event loop.
- **`read` blocks until bytes arrive.** The whole thread parks on a slow
  client. The event loop fixes that too — read whenever the socket says
  it's ready, otherwise tend to other connections.

These are the limitations of *blocking* servers, and you have built one.
Knowing what blocking servers can and cannot do is the prerequisite to
understanding why the next chapter is structured the way it is.

<div class="callout callout-experiment">

🧪 **Experiment**

Start the echo server, then open a second terminal and `nc localhost
6379` *without typing anything in the first*. Confirm the second
connection hangs in the kernel's listen backlog. Type something in the
first connection (now the server reads, writes back, and loops); your
second connection still hangs. Close the first (`Ctrl-D`); now the server
loops to `accept`, picks up the second, and you can chat with it. This
is exactly the limitation an event loop removes — the server should be
servicing *both* connections concurrently.

</div>

---

## Where this leaves Kestrel

`src/net/` has the three foundational types: `FileDescriptor` (RAII over a
POSIX fd), `Listener` (binds and accepts), `Connection` (reads and
writes). Every piece of Kestrel's networking from here forward composes
these, never re-implements them. The lessons:

- **Wrap every POSIX resource in an RAII class.** Sockets, files, mmaps,
  pipes, mutexes. Once the lifetime is in a destructor, the resource
  cannot leak, double-close, or live past its scope.
- **Convert errno into typed errors at the boundary.** Inside the wrapper:
  `errno`. Outside the wrapper: `std::expected<T, NetworkError>`. The
  conversion happens exactly once, where the syscall is called.
- **Loop on `EINTR`, on short writes, on short reads.** POSIX is allowed
  to do this; the C++ around it must not collapse.

Chapter 15 introduces non-blocking I/O and the **event loop** — one
process tending many connections by polling readiness. The interface will
be `kqueue` on macOS, `epoll` on Linux, hidden behind a single Kestrel
`EventLoop` class. Chapter 16 then ties the RESP parser from Chapter 6 to
the connection layer here, completing the read-side from "byte arrives on
socket" to "command parsed."

<div class="callout callout-ask">

💬 **Ask Claude**

> Here's my `net/` module (`file_descriptor.hpp`, `listener.{hpp,cpp}`,
> `connection.{hpp,cpp}`) and the echo server. Review: (1) is the RAII
> story complete — is there any path where a fd could leak or be
> double-closed? (2) am I handling `EINTR` and short writes correctly in
> `read`/`write_all`? (3) is `SO_REUSEADDR` enough for a robust server
> restart, or should I also set `SO_REUSEPORT`? (4) are my move
> operations correct, including self-move?

</div>

<div class="callout callout-checkpoint">

✅ **Checkpoint**

Build and tests/manual test green:

```bash
cmake --build src/build
./src/build/echo_server &
echo "hello" | nc localhost 6379         # should print "hello"
kill %1
```

You should be able to:

- Name the five syscalls of a Berkeley TCP server and what each does.
- Defend the `FileDescriptor` design: copy deleted, move provided,
  destructor closes, sentinel `-1`.
- Explain three socket bugs and how RAII prevents each.
- Describe what `SIGPIPE` is and Kestrel's mitigation.
- Predict what happens to a second client when the echo server is busy
  with the first, and explain why an event loop will fix it.

Hand over:

> Walk through my `net/` module end-to-end and look for fd-lifetime
> hazards. Where would a careless caller still leak or double-close a
> file descriptor? Are there `noexcept` annotations I'm missing on
> functions that genuinely cannot throw?

When that's clean, Chapter 15 turns these blocking primitives into a
non-blocking event loop.

</div>
