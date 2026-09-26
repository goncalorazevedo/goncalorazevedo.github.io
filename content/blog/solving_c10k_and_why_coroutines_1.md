+++
title = "Solving C10K (and what is async), Part 1"
date = 2026-08-26
series = "c10k"
taxonomies.tags = [
  "c10k",
  "async",
  "concurrency",
  "asyncio",
  "python",
]
+++

In 1999 (27 years ago!) Dan Kegel postulated the [C10K](https://www.kegel.com/c10k.html) problem:
>  It's time for web servers to handle ten thousand clients simultaneously, don't you think? After all, the web is a big place now.

The C10K appears with the popularization of the internet, back then people would like their servers to cope well with this new found fame. 
In this post series I will do a walkthrough (using Python) of server architectures and how they "evolved" to overcome the C10K problem. 

I will be building a KV store server with a very simple wire protocol.
If you have never come across the C10K problem and are not sure how it is solved today I highly encourage you to think about it before looking it up or reading this post. It's a fun exercise that shines a lot of light into many modern day aspects of programming.


---

## Protocol 

Our KV server relies on TCP. Clients, once connected, request a get or set operation.
The request is a simple fixed size string:
- `get <k>` to get the value for key "k"
- `set <k> <v>` to set the value "v" for key "k" 

The keys and values are always 1 byte characters.
For both operations the server returns the value of the associated key, for `set` it should return the newly set value.

## Naive servers

What we are aiming at is concurrency, we want clients to be served without having to line up on a queue. The simplest approach to achieve concurrency is via threads, delegating each connection to a newly created thread lets us write our logic in a procedural manner while the OS interleaves the client connections on the CPU transparently. 
```python
import socket
from threading import Thread

KV: dict[str, str] = {}

LISTEN_ADDR = ("0.0.0.0", 25000)

def server():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(LISTEN_ADDR)
    sock.listen()
    while True:
        conn, addr = sock.accept()
        Thread(target=handle_connection, args=(conn, addr)).start()

def handle_connection(conn: socket.socket, addr):
    print(f"Handling connection for {addr}")
    cmd = conn.recv(3) # assume recv call always return the requested bytes for simplicity
    match cmd:
        case b"get":
            # " k" should be in the buffer
            value = KV.get(conn.recv(2).decode()[1], "")
            conn.send(value.encode())
        case b"set":
             # " k v" should be in the buffer
            key = conn.recv(2).decode()[1]
            value = conn.recv(2).decode()[1]
            KV[key] = value
            conn.send(value.encode())

    conn.close()
    print(f"Closed connection for {addr}")

server()
```
It's simple to reason about this solution, but threads are not cheap. By today's standards we can probably handle 10k connections this way, but we should place ourselves in the context of the 90s: a 32-bit architecture with 1MB of thread stack memory cannot address the required 10GB.

What is still true today, however, is that relying on OS thread scheduling for concurrency adds unnecessary overhead, switching a running thread requires a dive into kernel space and we can probably achieve our concurrency needs more efficiently in userspace, it might also be inconvenient to have threads swapped by the OS at certain points.

Let's run this server under a PID capped cgroup to simulate 90s hardware and software limitations: 
```shell
systemd-run --scope --user -p TasksMax=2000 python3 server.py
```
```sh
for i in $(seq 1 3000); do
  (nc localhost 25000 <&- &) # no input to nc, just holds connection
done
```

```
Running as unit: run-p145915-i126195.scope; invocation ID: 8d4287debd89488d94561a452bdb5bf4
Handling connection for ('127.0.0.1', 49078)
[... more connection handling logs ... ]
Traceback (most recent call last):
  File "/home/goncalo/PycharmProjects/c10k/main.py", line 33, in <module>
    server()
    ~~~~~~^^
  File "/home/goncalo/PycharmProjects/c10k/main.py", line 15, in server
    Thread(target=handle_connection, args=(conn, addr)).start()
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^
  File "/home/goncalo/.local/share/uv/python/cpython-3.14.6-linux-x86_64-gnu/lib/python3.14/threading.py", line 1005, in start
    _start_joinable_thread(self._bootstrap, handle=self._os_thread_handle,
    ~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                           daemon=self.daemon)
                           ^^^^^^^^^^^^^^^^^^^
RuntimeError: can't start new thread
```
Farewell my beloved server 🕊️.

The next iteration of this idea is to cap the number of threads with a thread pool. 
In this enhanced version we delegate client connections to a fixed (or dynamic) number of threads and avoid having an uncontrollable amount spawned, that solves the previous issue.
```python
import socket

from threading import Thread
import queue

KV: dict[str, str] = {}

LISTEN_ADDR = ("0.0.0.0", 25000)

def server():
    work_queue = queue.Queue()
    for i in range(50):
        Thread(target=worker, args=(work_queue,), daemon=True).start()

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(LISTEN_ADDR)
    sock.listen()
    print(f"Listening on {LISTEN_ADDR[0]}:{LISTEN_ADDR[1]}")
    while True:
        conn, addr = sock.accept()
        work_queue.put((conn, addr))

def worker(work_queue: queue.Queue):
    while True:
        conn, addr = work_queue.get()
        handle_connection(conn, addr)

def handle_connection(conn: socket.socket, addr):
    print(f"Handling connection for {addr}")
    cmd = conn.recv(3) # assume recv call always return the requested bytes for simplicity
    match cmd:
        case b"get":
            # " k" should be in the buffer
            value = KV.get(conn.recv(2).decode()[1], "")
            conn.send(value.encode())
        case b"set":
             # " k v" should be in the buffer
            key = conn.recv(2).decode()[1]
            value = conn.recv(2).decode()[1]
            KV[key] = value
            conn.send(value.encode())
            conn.send(value.encode())

    conn.close()
    print(f"Closed connection for {addr}")

server()
```

This is an improvement, ignoring the unbound queue. We have solved the flaw of the first solution. However, what if all threads are handling a 
connection and a new one arrives?

```shell
python3 server.py &

for i in $(seq 1 50); do
  (nc localhost 25000 <&- &) # no input to nc, just holds connection
done
```
```shell
printf "get a" | nc localhost 25000 # get value from KV => infinitely waiting
```
It doesn't crash, but new clients aren't served. If the server was spinning the CPU at 100% then we would need to throw more metal at it, but that is not the case, the server is actually idle waiting for the messages on the connections! We need to find a more efficient solution.

## The C10K solution

Solving this architectural problem and optimizing the solutions is very important if you are anticipating C10K in the the 90s... 
There are tons of gems from those times, people were lowering web servers into the kernel, see 
[TUX](https://en.wikipedia.org/wiki/TUX_web_server) and [khttpd](https://www.linux.it/~rubini/docs/khttpd/khttpd.html).

If you know how C10K is solved this might seem crazy, but lowering the server into the kernel is not an outlandish idea. Even though these solutions might 
have been overkill for the web servers of the C10K era, what about the modern servers of the C10M and C10B era? People are still lowering applications into the kernel, 
for example [Netflix FreeBSD version](https://freebsdfoundation.org/end-user-stories/netflix-case-study/) with a custom sendfile replacement, [kTLS](https://docs.kernel.org/networking/tls-offload.html) or [eBPF](https://pt.wikipedia.org/wiki/EBPF) for high performance relays and proxies.

The answer to the C10K didn't end up requiring that much kernel level control, the solution was centered around a **userspace event-driven non-blocking IO architecture**. 
A lot of buzzwords for sure. [Flash Web Server](https://www.usenix.org/legacy/event/usenix99/full_papers/pai/pai.pdf)
was the closest paper from that era I could find describing this approach. 

Let's dissect all these buzzwords:
- *userspace* - not lowering the server into the kernel and not relying on OS scheduling. 
- *event-driven* - reacting to events. 
- *non-blocking IO* - not blocking threads on IO operations.

The idea is that one thread does not need to be bound to a single client connection, the lifetime of the request can be broken into
several steps, and the server can be thought of as a state machine that interleaves all requests's steps.
If everything was CPU bound this would not be relevant, because the interleaving of different steps would just be adding overhead
to the total computation, however, for servers, many of these steps involve potential blocking on IO,
so a thread can comfortably serve a client while another is in a blocked step of its request lifecycle. The decision on which request step to advance next is driven by events on the corresponding connections.

Looking at a 90s web server, the lifecycle of a request is broken into accepting a connection, reading the message, parsing the message, finding a file, sending the headers and sending the file.
Almost all of these steps are potentially blocking.

That's the theory, in the [next post](../solving-c10k-and-why-coroutines-2) I will try to implement it!


