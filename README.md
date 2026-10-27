# Multi-Threaded Server

A calculator server and client in C, using TCP sockets, a **thread pool** (pthreads) and **semaphores**.

Clients send simple math problems like `7 * 6`. The server solves each one in a worker thread and sends the answer back. The server has only **3 worker threads**, so when more than 3 requests come in at once, the extra ones wait in line until a thread is free. The server prints timing logs, so you can see this waiting happen.

## How it works

**Server** (`server.c`, port `5002`):

1. At start, it creates a pool of **3 worker threads**. Each worker sleeps on its own semaphore until it gets a job.
2. The main thread accepts connections. For each one, it waits until a worker is free (a semaphore that counts free workers), then gives the connection to that worker.
3. The worker reads the request, calculates the result, and **waits 20 seconds** (to simulate slow work).
4. The worker then **connects back to the client** on the port the client gave in its request and sends the result.
5. The server prints, for each client: when it connected, how long it waited for a thread, the question, and how long the answer took.

**Client** (`client.c`):

1. A background thread opens a listening socket on a random free port. The server sends answers there.
2. The main loop asks for a math problem, then sends `127.0.0.1 - <listening port> - <problem>` to the server at `127.0.0.1:5002`.
3. The client doesn't wait for the answer. You can send more problems right away, and answers are printed when they arrive.

**Supported problems:** two integers and one operator: `+`, `-`, `*`, `/` (integer division), `%`. Examples: `7 * 6`, `17 % 5`, `10/2`.

## Requirements

- **Linux** (or WSL on Windows). The code uses POSIX sockets, pthreads and features that macOS doesn't have (pthread barriers, unnamed semaphores).
- **GCC**

## How to run

Open two or more terminals in the project folder.

**Terminal 1: start the server**

```bash
bash run_server.sh
```

This compiles `server.c` to `server.out` and runs it. You should see `Server is listening...`.

**Terminal 2 (and more): start a client**

```bash
bash run_client.sh
```

This compiles `client.c` to `client.out` and runs it. At the `Enter you sentence:` prompt, type a problem and press Enter:

```
Enter you sentence: 7 * 6
```

About 20 seconds later the client prints:

```
server socket started
request was answered in 20.000000 seconds
the result is: 42
```

Stop the server and clients with `Ctrl+C`.

**Don't use `make`.** The Makefile runs the server and then the client in the same terminal, but the server never stops, so the client never starts.

## Try the thread pool

Send 4 problems quickly (for example, 2 from each of two clients). The server log shows the first 3 getting a thread right away, and the 4th waiting:

```
client 127.0.0.1:41812 -> took 19.000000 seconds for server to assign a thread
...
client 127.0.0.1:41812 -> took 39.000000 seconds for server to respond
```

To change the behaviour, edit these values in `server.c`:

| Setting | Value | Meaning |
|---|---|---|
| `MAX_THREADS` | 3 | Number of worker threads |
| `QUEUE_SIZE` | 20 | Waiting-connection queue size for `listen()` |
| `sleep(20)` | 20 s | Simulated work time per request |
| `portno` | 5002 | Server port (also set in `client.c`) |

## Known issues

- **GCC 14 or newer can't compile the client:** `implicit declaration of function 'gets'`. Newer C standards removed `gets`. Fix: add `-std=gnu99` to the `gcc` line in `run_client.sh`:

  ```bash
  gcc -std=gnu99 -o client.out client.c -lpthread -w
  ```

- **`x % 0` crashes the server** (floating point exception). Division by zero (`x / 0`) doesn't crash; it returns `2147483647` (INT_MAX).
- **Restarting the server right away** may fail with `ERROR on binding: Address already in use`. Wait a few seconds and try again.
- **Everything runs on one computer:** the client always connects to `127.0.0.1`.
- **The `server` and `client` files in the repo are older builds**, not from the current code. The run scripts build fresh `server.out` and `client.out` instead.

## License

[MIT](LICENSE)
