# webserv — Project Plan

---

## 1. Project overview

`webserv` is an HTTP server written in C++98, started with a configuration file:

```
./webserv [configuration file]
```

The configuration file defines the interface and port pairs the server listens
on, the routes it serves, and the rules applied to each route. The server
accepts connections from standard web browsers, serves static content, accepts
file uploads, deletes resources, and executes CGI programs.

All input/output on sockets and pipes is non-blocking and driven by a single
`poll()` (or equivalent) call for the entire program. The server must remain
operational at all times: it must never crash, never terminate unexpectedly, and
never leave a request hanging indefinitely.

---

## 2. How the server operates

### Startup

1. The configuration file is read and validated, producing the list of servers,
   their listening addresses, and their per-route rules. *(Part 3)*
2. A listening socket is created and bound for each configured address, set to
   non-blocking mode, and registered with `poll()`. *(Part 1)*
3. The server enters its main loop and waits.

### Handling a request

1. `poll()` reports a listening socket as ready. The connection is accepted,
   set to non-blocking, and registered. *(Part 1)*
2. `poll()` reports a client socket as readable. The available bytes are read
   into that connection's buffer. *(Part 1)*
3. The buffer is passed to the request parser, which reports either that the
   request is incomplete — in which case the connection waits for more data — or
   that it is complete, returning the method, target, version, headers and
   body. *(Part 2)*
4. The parsed request is resolved against the configuration: the matching route
   is identified, the method is checked against the methods permitted on that
   route, and the target is translated into a filesystem path. The requested
   operation is then performed — reading a file, writing an uploaded file,
   deleting a resource, or returning a redirection. The result is a status code
   and, where applicable, content and a content type. *(Part 3)*
5. If the route maps the target's extension to a CGI interpreter, step 4 is
   replaced by CGI execution: the interpreter is launched, the request body is
   written to its standard input, and its output is collected. *(Part 1)*
6. The status code and content are serialized into a complete HTTP response:
   status line, headers, empty line, body. *(Part 2)*
7. `poll()` reports the client socket as writable and the response is written.
   A response that cannot be written in full is resumed on subsequent write
   readiness. *(Part 1)*
8. The connection is either retained for a further request or closed. *(Part 1)*

### Relationship between the parts

Part 1 owns the program's control flow. It invokes Part 2 and Part 3 and
receives their results. Part 2 and Part 3 do not invoke one another, hold no
connection state, and are written to process a single request at a time.
Concurrency is handled entirely within Part 1.

---

## 3. Division of work

### Part 1 — Connections and process management — **NAVANEETHA**

Owns the main loop, all socket and pipe I/O, and all child processes.

- Creating, binding and listening on a socket for every configured address
- Setting all sockets and pipes to non-blocking mode
- A single `poll()` monitoring all listening sockets, client sockets and CGI
  pipes, for both read and write readiness
- Accepting incoming connections
- Reading request data, accumulating it per connection, and submitting it to the
  parser until a complete request is available
- Invoking the request handler and the response serializer, in order
- Writing responses, including responses requiring multiple write operations
- Connection lifecycle: creation, reuse for subsequent requests, and teardown
- Timeouts on idle and incomplete connections, so that no request hangs
  indefinitely
- Handling client disconnection at any point without disrupting the server
- Ignoring `SIGPIPE`
- CGI execution: constructing the environment, creating pipes, `fork()` and
  `execve()`, changing to the script's directory, writing the request body to
  the child, reading its output until end of file, reaping the child without
  blocking, and terminating a child that exceeds its time limit
- Ensuring `fork()` is not used anywhere else in the project
- Stress testing the completed server under concurrent load

### Part 2 — HTTP protocol — **SUPRIYA**

Owns the interpretation of request data and the construction of responses.
Performs no I/O.

- Parsing the request line: method, target, query string and HTTP version
- Parsing headers into name/value pairs, with limits on count and total size
- Parsing the body according to `Content-Length`
- Decoding chunked transfer encoding into a complete body
- Parsing `multipart/form-data` bodies into their individual parts, so that an
  uploaded file's name and contents can be handed to Part 3
- Supporting incremental parsing, so that a request delivered across multiple
  reads is accumulated and reported as incomplete until it is whole
- Resetting parser state between requests on a persistent connection
- Determining the correct status code for malformed, unsupported or
  disallowed requests, including 400, 405, 411, 413, 414, 431, 501 and 505
- Serializing responses: status line, headers, empty line, body
- Default error pages, used when the configuration specifies none
- Persistent connection semantics, including honouring `Connection: close`
- Verifying our responses and headers against NGINX for equivalent requests

### Part 3 — Configuration and resources — **URSLA**

Owns the configuration file and all filesystem access.

- Parsing the configuration file, including nested server and location blocks
- Supporting the directives required by the subject: listening address and port,
  custom error pages, maximum client body size, and per route: permitted
  methods, HTTP redirection, root directory, directory listing, default file,
  upload permission and storage location, and CGI interpreter by file extension
- Supporting multiple server blocks on multiple ports
- Validating the configuration and exiting with a clear diagnostic on error
- Falling back to a default configuration path when no argument is supplied
- Matching a request target to its route and translating it to a filesystem
  path according to the route's root
- Verifying the requested method is permitted on the matched route
- `GET`: locating and reading files, and determining the content type
- Directory targets: serving the configured default file, generating a directory
  listing, or refusing
- `POST`: storing uploaded files in the configured location
- `DELETE`: removing the target resource and reporting the outcome
- Applying configured redirections
- Enforcing the maximum client body size
- Rejecting targets that traverse outside the route's root

---

## 4. Project structure

Each part of the project lives in its own directory. A directory contains
everything belonging to it: its sources, its header, and its own Makefile.

```
webserv/
├── Makefile                 shared — links every module into the final binary
├── README.md
├── PLAN.md
│
├── includes/                shared — the headers the parts use to call each other
│   ├── Config.hpp
│   ├── HttpRequest.hpp
│   └── HttpResponse.hpp
│
├── src/                     Part 1 · NAVANEETHA — entry point and main loop
│   ├── Makefile
│   ├── server.hpp
│   ├── main.cpp
│   ├── server/              sockets, poll loop, connections
│   └── cgi/                 CGI processes
│       └── cgi.hpp
│
├── http/                    Part 2 · SUPRIYA — request parsing, response building
│   ├── Makefile
│   └── http.hpp
│
├── config/                  Part 3 · URSLA — configuration file and routes
│   ├── Makefile
│   └── config.hpp
│
├── resources/               Part 3 · URSLA — files, uploads, deletions
│   ├── Makefile
│   └── resources.hpp
│
├── configs/                 configuration files for testing and evaluation
└── www/                     the demonstration website
```

**Building**

- Running `make` inside a module directory compiles that module on its own, so
  each of us can build and check our own part without building the project.
- The top-level Makefile calls each module's Makefile and links the results
  into `webserv`. It provides `NAME`, `all`, `clean`, `fclean` and `re`.
- The top-level Makefile is shared. Each owner adds their own module to it;
  nobody waits for somebody else to get their code building. It is written so
  that each module appears in exactly one place, to keep conflicts rare.

**Rules**

- `includes/` holds only the headers needed by more than one module. These are
  the interfaces between the three parts, and they are what the parts use to
  call each other. A header used by a single module stays inside that module's
  directory, and the other parts never include it.
- Each directory has exactly one owner. Nobody modifies a directory they do not
  own.
- **The files inside a directory are decided by its owner, as the work
  proceeds.** The directory structure is fixed; the list of files is not. No
  one should expect to know their file list before they start writing.

---

## 5. Git workflow

- `main` always compiles. Nothing is committed to it directly.
- **One branch per module, named after the module** — for example `poll-loop`,
  `request-parser`, `config-lexer`, `upload`, `cgi-exec`. Branches are not
  named after people, and nobody works on a single long-running personal
  branch.
- A branch covers one module. If work on it starts touching a second module, it
  is split into two branches.
- A branch must be small enough for the other two to read in one sitting. If it
  is not, it should have been two branches.
- Branches are opened, finished and merged promptly. A branch is not held open
  for a week.
- Before merging: pull `main`, confirm the project still compiles with
  `-Wall -Wextra -Werror -std=c++98`, then open the merge.
- A branch is merged only after the other two members have read it.
- Any change to a header in `includes/` goes on its own branch and is agreed by
  all three before it is merged, because it affects every part.

**Commit messages**

- Format: `<module>: <what changed>` — for example
  `poll-loop: handle partial writes`, `config: reject duplicate listen
  directives`, `request-parser: accept headers split across reads`.
- One commit is one change. A day's work is not a single commit.
- `fix`, `update`, `wip` and `test` are not commit messages on their own.
- The message does not name the author. Git records who made the commit.

**When one part has to be changed by somebody else**

Connecting the parts will sometimes require a change inside a directory you do
not own. This is expected and it is allowed, under these conditions.

- The change goes on its own branch, named for what it is — for example
  `integration-http-server` — and not mixed into a module branch.
- Only what is needed to connect the parts is changed: adding a helper file,
  adjusting a function signature, correcting an include. The logic inside the
  other part is not modified. If the change would alter how that part behaves,
  it is requested from its owner instead of being made directly.
- The owner of that directory reviews and merges it. They are not told
  afterwards.
- Where the mismatch is between two parts rather than inside one, the correct
  change is usually to the shared header in `includes/` rather than to anyone's
  internals.

---

## 6. Completion criteria

The project is complete when all of the following hold.

- The program does not crash or terminate unexpectedly under any circumstances
- The Makefile provides the required rules and performs no unnecessary relinking
- The project compiles with `c++ -Wall -Wextra -Werror` and with `-std=c++98`
- No external libraries and no Boost
- A configuration file is accepted as an argument, and a default path exists
- All socket and pipe I/O is non-blocking and driven by one `poll()` call
- `poll()` monitors reading and writing simultaneously
- No read or write occurs without prior readiness, and `errno` is not inspected
  after a read or write
- `fork()` is used for CGI only
- `GET`, `POST` and `DELETE` are implemented
- Clients can upload files
- A fully static website is served correctly
- Status codes are accurate
- Default error pages exist
- The server listens on multiple ports and serves different content on each
- No request hangs indefinitely
- The server is compatible with standard web browsers
- The server remains available under stress testing
- `README.md` meets the requirements of chapter V

The bonus part is assessed only if the mandatory part is complete without
issues. If undertaken, cookies and session management takes priority over
support for multiple CGI types.
