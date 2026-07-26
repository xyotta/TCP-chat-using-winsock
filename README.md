# TCP chat
Multithreaded client-server TCP chat, that i built to get a hands-on networking
and some modern C++. Raw winsock2 sockets, no external libraries. Leaned into multithreaading instead of multiplexing, tried not to make code seens C-ish.

## Origin
Started from Microsoft's guide on getting single client talking with server:
https://learn.microsoft.com/en-us/windows/win32/winsock/windows-sockets-start-page-2
Then i modified logic and restructured it. When i looked into *'how to get more clients'* - usual answer is `poll()` / `select()` - last one i already used in a similar C project on Linux(Ubuntu), so i thought to try thread-per-client approach. This approach is really heavy on resources, at least heavier than multiplexing, but i wanted to see how far threading primitives could take it, and it ended up reading a lot simpler than if it were manualy polling loop.

## Architecture 

### Hierarchy 

Socket_base
├── Socket_server
└── Socket_client

**Socket_base** - is an RAII wrapper around WSA and socket init/teardown. It tracks socket count via static variable to provent `WSACleanup()` from firing while some socket still active.
Shared state ( `socket_fd` , `result` `hints`) i made them *protected* to get access to them from derived classes.

**Socket_server** - binds,listens and accepts on it's own `std::jthread`. Each connected client becomes a `Session_unit` that stored in a `std::vector<std::unique_ptr<Session_unit>>` and ptotected by `std::mutex`. Dead sessions are cleaned whenever a new client connects. Incoming messages are broadcasted to all other sessions.

**Socket_client** - connects to server. Runs a receive loop on a background using `std::jthread` while the main blocks on `std::getline` for user input.

**Session_unit** - represents one connected client on server side. Owns a `std::jthread` that blocks on `recv()` so `unique_ptr` storage needed since thread cannot be copied. Mark itself `is_dead` when the client disconnects.

## Design decisions
**Synchronous Thread-per-Client Model**
While select()/poll() require a manual event loop that checks every file descriptor on each iteration to determine which sockets are ready, my model simplifies server implementation. By dedicating a single thread to each session, every connection uses an independent, blocking (synchronous) recv() call. This keeps the code entirely linear, leaving the complex job of interleaving execution to the OS scheduler.

The tradeoff for this clean logic is memory consumption. Since each thread requires approximately 1MB of stack space, this approach inherently breaks down once you scale past a few thousand connected clients. For a project focused on learning, this is a completely acceptable boundary. In fact, hitting that exact scalability wall is the best way to practically understand why true asynchronous I/O architectures exist.
