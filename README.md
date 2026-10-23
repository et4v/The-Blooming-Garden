<p align="center">
  <img src="assets/lily.png" alt="A pink lily" width="220">
</p>

# 🌸 The Blooming Garden 🌸
A Mini-Distributed Control System in C

## Overall Concept
I built a small simulated "garden" where every flower is its own networked program. Each flower connects to a central server over TCP and has between one and eight petals that move gradually toward their targets.

It's structured like a supervisory controller and its embedded nodes: the server sends high-level commands ("open," "run sequence 2") to every flower or just one, and each flower runs its own motion loop and reports its status back. That's the same split a main controller and small motor boards use, where the boards handle their own timing while listening for commands.

---

## Client Responsibilities
Each client program is a single flower. After connecting to the server, each flower:

- Introduces itself with a `HELLO` so the server can register it by name
- Runs a motion loop every 100 ms, moving each petal a little closer to its target angle
- Sends `STATUS` messages back to the server with its state (`IDLE` or `MOVING`) and petal angles
- Prints its own petal angles so you can watch it move
- Closes its petals and exits cleanly when it receives `TERMINATE`

---

## Server Responsibilities
The server is the central controller for the garden. It:

- Accepts up to 64 flowers at once, one thread per connection
- Keeps a table of connected flowers (name, connection, latest status), protected by a mutex
- Caches each flower's most recent status so you can check on the whole garden at any time
- Runs an interactive `garden>` console for sending commands (case-insensitive)
- Shuts down gracefully: `QUIT` closes every flower, terminates them, and waits for all of them to disconnect before exiting

### Commands

| Command | What it does |
|---|---|
| `OPEN all\|<name>` | Open every flower, or one flower |
| `CLOSE all\|<name>` | Close every flower, or one flower |
| `SEQ1 all\|<name>` | Petal sequence 1 (left to right) |
| `SEQ2 all\|<name>` | Petal sequence 2 (outside in) |
| `TERMINATE all\|<name>` | Close and disconnect flowers |
| `BLOOM` | Every flower gets a random sequence, with randomized staggered timing |
| `LIST` | List connected flowers |
| `STATUS` | Show each flower's most recent status |
| `HELP` | Show the command list |
| `QUIT` | Close and terminate every flower, then shut down |

---

## Why the Files Are Split
Petal motion, angles, and sequences live in `flower.c` and `flower.h`. Networking and threading live in the server and client files.

`flower.c` is plain, portable C: it uses only the standard library (no threads, sockets, or OS calls). If the garden were ever built with real hardware, the flower logic could move to a microcontroller unchanged, with only the communication layer swapped out.

---

## Built With
- **Language:** C
- **Networking and concurrency:** POSIX sockets and pthreads, through the CS:APP wrapper library
- **Build system:** Makefile (builds both the server and the client)

---

## How to Run

### macOS / Linux
1. Make sure you have a C compiler (`gcc` or `clang`) and `make`.
2. **Get the CS:APP library:** download `csapp.c` and `csapp.h` from the [CS:APP code examples page](https://csapp.cs.cmu.edu/3e/code.html) and put both files in the project folder.
3. Build both programs:
   ```bash
   make
   ```
4. Start the server first:
   ```bash
   ./garden_server <port>
   ```
5. In separate terminals, start one or more flowers (1 to 8 petals each):
   ```bash
   ./flower_client <server_host> <port> <flower_name> <num_petals>
   ```
6. Type commands into the server's `garden>` prompt.

For example, a three-flower garden:
```bash
./garden_server 5050
./flower_client localhost 5050 rose 5
./flower_client localhost 5050 tulip 6
./flower_client localhost 5050 daisy 8
```
Then type `BLOOM` at the `garden>` prompt and watch each flower's terminal.

### Windows
The code uses POSIX sockets and pthreads, which Windows doesn't provide natively. Run it in Windows Subsystem for Linux (WSL), or another Unix-compatible environment such as MSYS2.

### Running Across Multiple Devices (So Cool)
The server and flowers don't have to be on the same computer:

- Pick one device to be the **garden server** and start it first
- Find that device's **IP address or hostname**
- On each flower device, pass the server's IP address as `<server_host>`
- Start the flowers only after the server is running

Notes:
- All devices need to be on the same local network, or otherwise able to reach each other
- If flowers can't connect, check firewalls first; the server's port has to be reachable

---

## Known Limitations
Things I'd fix next, and what I'd change for real hardware:

- **Message framing:** TCP is a byte stream, so one `read()` can return part of a message or two messages at once. The code currently treats each read as one message; a buffered line reader would fix it.
- **Disconnects:** writing to a flower that has already disconnected can raise `SIGPIPE`, which stops the server.
- **Partial writes:** `write()` can send fewer bytes than requested, and the code doesn't retry the rest.
- **Scaling:** one thread per flower is simple but heavy at large scale; an event loop (`poll`/`epoll`) would scale further.
- **Real hardware:** physical boards would use a bus or lightweight protocol instead of TCP, a hardware timer instead of `sleep`, heartbeats with a safe state when the controller disappears, and sensor feedback instead of computed petal positions.
