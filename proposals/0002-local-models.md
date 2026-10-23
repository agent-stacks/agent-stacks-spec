# Proposal 0002: Running a stack's agent client against a local model

- Status: Draft, for discussion. Experimental; like everything before
  1.0.0, anything here may change or be withdrawn.
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.3, and the sections it
  would touch: §5.1, §6.1, §6.4, §7, §8, §9, §10. Composes with
  [Proposal 0001](0001-host-sandbox.md)
  ([#4](https://github.com/agent-stacks/agent-stacks-spec/pull/4)).
- Scope: any number of models per stack. A carried model is served on
  the host where the stack runs, and a borrowed one wherever it is
  already served. Designed for two use cases. Everything else is listed
  under [Non-goals](#non-goals).

## Summary

Today a stack can't name the model its agent client talks to. The
agent client uses its own default, which is usually its vendor's hosted
API. This proposal adds models to a stack, either carried in the stack
or borrowed from a server that is already running.

Four parts, each small enough to change later:

1. **Layout.** A stack MAY hold models under
   `<stack root>/share/agent-models/`, one directory per model. A
   carried model's directory holds its weights, the server that runs
   them, and a `bin/serve` executable that joins the two. A borrowed
   model's directory holds an `endpoint.json` saying where the model is
   served.
2. **Build.** Weights are a pinned source, fetched when the stack is
   built. The declaration names the server, and the builder resolves
   and links it the way §8 resolves a plugin's runtimes, failing the
   build when it can't.
3. **Declaration.** The vocabulary in §6.1 gains `models`, each carried
   or borrowed per §6.2, and `defaultModel`, the one the agent client
   starts with.
4. **Launch.** The stack owns the lifecycle of every server it starts.
   Launching the stack starts them on loopback ports and gives each
   model to the agent client. Once the last agent client the stack
   started has exited, nothing the stack started for its models is
   still running. When a declared model can't be served, the launcher
   refuses to launch.

No new weights format and no new wire protocol are proposed. What else
the agent client can reach is left to the sandbox
([Proposal 0001](0001-host-sandbox.md)).

## Motivation

### Two use cases

This proposal is designed against two use cases, and should be judged
by how well it serves them.

**Data that does not leave the machine.** A person or a team wants an
agent to work on data they will not send to a third party: medical or
financial records, a private codebase, personal notes and measurements.
They will accept a less capable model to keep the data on their own
machine.

**Compliance where the data lives.** An organization holds data under
rules that restrict where it may be processed, and wants an agent on
hardware already inside those rules, possibly with no network at all.
It needs to install the whole thing offline, and to show someone else,
before it runs, what it is and where it can send data.

### What they need

1. **The model travels with the stack.** Pinned by digest, installed
   with everything else, never downloaded on first run, so that the
   stack runs on a machine with no network. This is §5.1(6) applied to
   one more thing.
2. **The stack starts and stops its servers.** Nothing has to be
   installed or started before the stack runs, and nothing the stack
   started is still running after it exits. A server outside that
   lifecycle is outside the unit a consumer can inspect.
3. **A declared model answers, or nothing does.** When the model can't
   be served, the agent client doesn't run against something else in
   its place.
4. **Inspectable before it runs.** Which models, what serves each one,
   and where.
5. **Enforcement comes from a sandbox.** The declaration decides which
   models the agent client is given. Only a sandbox decides what else
   it can reach.

The first use case needs 3 and 5. The second needs all five.

### Models served elsewhere

Open models are increasingly run with popular agent clients from
somewhere other than the consumer's machine: a GPU server on the
organization's network, inference running in its own cloud account, a
provider that hosts open models. The same declaration should cover
those, so that a stack can say which model its agent client uses
wherever that model is served.
[Deployment patterns](#5-deployment-patterns) shows how.

### What a local model needs

Running a model locally takes three layers:

| Layer | Examples | Carried? |
|---|---|---|
| Weights | a GGUF file, an MLX model directory | Yes, pinned by digest like any other source |
| Server | llama.cpp, Ollama, oMLX, vLLM | Yes, named by the declaration and resolved like a runtime (§8) |
| Host | an accelerator, its driver, memory | No |

The first two are what this proposal adds to a stack. The third is what
a stack built to it still asks of the machine, and
[What the stack cannot carry](#3-what-the-stack-cannot-carry) says how
a launcher handles it.

### What it optimizes for

- **Start small.** One directory per model, one executable to serve
  it, one file to point at it. Each can grow later without undoing
  what is here.
- **The framework, not today's models.** The proposal names no model,
  parameter count, quantization, context window, or hardware. Those
  change every few months. What a stack has to say about a model does
  not: where its weights are, what serves them, where the agent sends
  its prompts, and what happens when that fails. A stack built to this
  proposal behaves the same whether the model it carries is good
  enough for its task this year or only next.
- **Two use cases, served well.** A stack can name several models,
  local and hosted, so cost savings and workflows that mix the two can
  be built on this. They are not what it was tested against, because
  both depend on how capable today's local models are, which is what
  the previous point declines to design around.

### Why it belongs in the stack

§5.4 describes everything a stack holds as configuration around one
agent. The model is the most consequential piece of that configuration
and the only one a stack cannot express today: whatever its author
intended, the stack sends what its agent reads to the agent's vendor.
§11.3 lists sandboxing and secrets as the axes already visible. This
is a third, and it composes with both: a carried model needs no API
key, which is one less secret to place, and a sandbox is what turns
"configured to use a local model" into "unable to use anything else."

## Non-goals

- **Choosing a model.** Which model suits a task or a machine is the
  author's or a tool's decision. A stack records the choice; it does
  not make it.
- **Deciding which model answers a request.** A stack can give its
  agent client several models. Which one handles a given request is
  the agent client's choice, through its own model switcher or
  settings; Claude Code, for one, already uses a smaller model for
  background work. The stack does not route: local first with
  escalation to a hosted model, a planner and an executor, a policy
  based on cost. A router can still be used, either as a borrowed
  endpoint or as the agent client of an outer stack whose inner stacks
  each carry a model (§5.4).
- **Restricting where data goes.** Whether the agent client can reach
  its vendor's API, a remote MCP server, or anything else is the
  sandbox's decision, made by its egress policy (Proposal 0001). This
  proposal decides which models the agent client is given, not what
  else it can reach. A stack that must send nothing off the host
  carries its models and a sandbox that denies everything else.
- **Starting a model on another machine.** A carried model is served
  on the host the stack runs on. A model served elsewhere is borrowed,
  and starting and stopping it is the job of whoever runs it. A stack
  moved to another machine, such as a GPU workstation or a VM, carries
  its models there and serves them there.
- **One server shared between stacks.** A server serves the stack that
  started it. Concurrent launches of that stack may share it
  ([Launch](#6-launch)); another stack may not. A server the consumer
  runs for everything is reached as a borrowed model.
- **Models a plugin uses.** An embedding model behind a plugin's MCP
  server is that plugin's runtime, carried as §8 carries any other.
  This proposal is about the models the agent client itself talks to.
- **How weights are distributed.** Registries, mirrors and transfer are
  the builder's concern. The proposal asks only that weights be pinned,
  carried like any other source, and stored once ([Build](#2-build)).
- **Context management.** How much a smaller model can do depends
  heavily on how the agent client manages its context: compacting it,
  handing work off, checking results. That is the agent client's
  concern, or a plugin's.

## Proposal

### 1. Layout

1. A stack MAY hold models at
   `<stack root>/share/agent-models/<model-name>/`, one directory per
   model, as many as it uses. Model names SHOULD be unique within a
   stack.
2. A carried model's directory SHOULD hold the model's weights and a
   `bin/` holding an executable named `serve` beside a link to the
   server it runs. The launcher reads neither the weights nor the
   server. It runs `serve`.
3. A borrowed model's directory SHOULD hold one file, `endpoint.json`,
   and nothing to run.
4. The weights and the server SHOULD be links into the stack's
   dependency set, as a plugin's runtimes are (§8.3). In a
   content-addressed store, the path a link resolves to names the
   digest of what it points at, which is how a consumer can see which
   weights and which server a stack carries without a manifest.
5. `<model-name>` is how the stack refers to the model. For a carried
   model it is also the name the agent client requests.

```text
caveman-stack/
├── bin/
│   └── caveman-stack
└── share/
    ├── agent-plugins/
    │   └── caveman/...
    └── agent-models/
        ├── qwen3-8b/                  carried
        │   ├── bin/
        │   │   ├── serve              the serve contract
        │   │   ├── check              optional host check
        │   │   └── llama-server       link to the server it runs
        │   └── qwen3-8b-q4_k_m.gguf   link to the weights
        └── team-coder/                borrowed
            └── endpoint.json
```

This is the move §5.1 makes for plugins and Proposal 0001 makes for
sandboxes: a directory under `share/` per thing, discoverable by
listing it. Listing `agent-models/` shows every model the stack can
give its agent client, and whether each is carried (`bin/serve`) or
borrowed (`endpoint.json`). A stack with no `agent-models/` directory
runs exactly as it does today.

#### `serve`

`serve` is the whole interface between a launcher and a carried model.
The launcher runs it as:

```sh
<model dir>/bin/serve --listen 127.0.0.1:<port> --protocol <name>
```

Those are its only arguments. Everything else, from the weights' path
to the server's flags and the context length it allocates, is fixed in
`serve` when the stack is built. The files on disk therefore say
exactly how the model will be served. A launcher needs to know nothing
about the server, so supporting a new server is work for the builder
and none for the launcher.

`serve`:

- serves the model at that address, over that protocol, answering
  requests that name `<model-name>`, in the foreground, until it
  receives `SIGTERM`;
- writes exactly one line to standard output: `ready` once the model
  is loaded and answering, or `error <code> <message>` when it cannot
  serve, after which it exits non-zero;
- writes everything else, including the server's own log, to standard
  error.

The code is one of:

| Code | Meaning |
|---|---|
| `protocol` | the server does not speak the protocol asked for |
| `address` | the address cannot be listened on |
| `weights` | the weights are missing or will not load |
| `memory` | the host does not have the memory the model needs as built |
| `host` | the host lacks something else the server needs: an accelerator, a driver, an operating system version |
| `other` | anything else; the message says what |

The codes are for whatever drives a launcher, such as an editor, a CI
job, or a catalog of stacks, which can act on `memory` without reading
a server's log. The message is for a person.

In practice `serve` is a short script the builder generates. For the
model above, its core is one command:

```sh
"$here/llama-server" --model "$here/../qwen3-8b-q4_k_m.gguf" \
  --alias qwen3-8b --ctx-size 32768 \
  --host "$host" --port "$port" >&2 &
```

Around it, the script parses its two arguments, refuses a protocol
llama-server does not serve, waits for the server's `/health` endpoint
to answer before printing `ready`, and passes `SIGTERM` on to the
server.

#### `check`

A model too large for the host often fails only after a long load. A
carried model's directory MAY also hold `bin/check`, which takes
`--protocol <name>`, loads nothing, and writes one line to standard
output: `ok` when the host can serve the model as built, or the same
`error <code> <message>` line `serve` would write. When it is present,
a launcher SHOULD run it before `serve`, and a consumer or a tool can
run it without launching anything. When it is absent, the same failure
arrives later, from `serve`, in the same form.

#### `endpoint.json`

A borrowed model's directory holds:

```json
{
  "url": "https://models.example.com",
  "model": "qwen3-coder-30b-a3b",
  "protocols": ["anthropic-messages", "openai-chat-completions"]
}
```

- `url` — the base URL the model is served at. A protocol's paths
  (Appendix A) are appended to it.
- `model` — the name to request there.
- `protocols` — the protocols the endpoint speaks, by the names in
  Appendix A.

`endpoint.json` SHOULD NOT hold a credential. A stack is meant to be
copied and inspected, and a credential in it travels with every copy.
How an endpoint that needs one gets it is open question 5.

### 2. Build

A model's weights are a source like any other, and §10.1 applies: a
builder SHOULD pin them by digest and fetch them when the stack is
built, never when it is launched. A stack that downloads its model the
first time it runs is not transportable (§5.1(6)), cannot run where
there is no network, and leaves the download behind when it is gone.

Weights are much larger than other sources: gigabytes, where a plugin
is often kilobytes. That changes how long a build takes and how much a
copy weighs, not what a build does. Two recommendations keep the cost
to one copy per machine:

- A builder SHOULD store weights once per digest, so that every stack
  carrying the same model links to the same file.
- A launcher SHOULD NOT copy weights at launch. Appendix B.4.3 makes
  the same case for staging plugins as links.

How a builder fetches them (resuming, mirrors, showing progress) is up
to the builder.

The declaration names the server. A format admits several servers: a
GGUF file runs on llama.cpp or Ollama, an MLX model on oMLX or mlx-lm.
Servers differ in the protocols they speak, the flags they take, and
how they use the same hardware, so the choice belongs in the
declaration, and on disk, where a consumer can see it.

1. A builder SHOULD resolve the named server to a package and link it
   into the model directory, as §8.3 links a plugin's runtimes, so that
   the stack does not depend on what the consumer has installed.
2. A builder that cannot resolve the named server SHOULD fail the
   build, naming the model and the server, as §8.4 asks for a runtime.
   So SHOULD a builder that knows the server cannot load the weights'
   format.
3. A builder SHOULD generate `serve`, and `check` where it can, and fix
   every serving parameter in them: the weights' path, the context
   length, offload, concurrency. `serve` is to a model what a rewritten
   shebang is to a script: the builder's glue between content and the
   runtime it needs.
4. For a borrowed model, a builder SHOULD write `endpoint.json` from the
   declaration, and fetch nothing.

Changing a serving parameter means building the stack again. A launcher
passes only an address and a protocol, so the stack's files stay a
complete record of how each model is served.

### 3. What the stack cannot carry

The server, and the user-space libraries it links against, are carried
like any runtime. Some of what it needs cannot be: the accelerator,
its kernel driver and any user-space driver library that has to match
that driver, an operating system version the server requires, and
enough memory to hold the model.

For a stack that carries a model, §5.1(6) therefore reads: copying the
stack and its dependency set to a machine with no server installed
yields a working stack, provided that machine has the accelerator and
driver the server uses.

When `check` or `serve` fails with `memory` or `host`, the launcher
SHOULD report it in those terms, with the message, rather than leave
the consumer to read a server's log.

The proposal does not say how much memory or which accelerator a model
needs. `check` lets a model's author test for both on the host without
the specification having to name hardware.

### 4. Declaration

§6.1 gains two optional fields:

- `models` — the models the agent client can use, each declared as
  below.
- `defaultModel` — the name of the model the agent client starts with.
  A stack with one model starts with that one. A stack with several
  SHOULD name one, and a builder SHOULD fail the build when it does
  not, rather than pick.

Each model has a `name`, which is its directory under
`share/agent-models/`. §6.2 applies, and a declaration SHOULD say which
form a model takes rather than leave that to which fields are present:

- **Carried.** The model is part of the stack's dependency set, and the
  stack serves the same model on every machine it reaches, including
  one with no network. A carried model is declared by:
  - `source` — where the weights come from, pinned by revision and
    content digest.
  - `server` — the package that serves them, named as the builder
    names packages.
- **Borrowed.** The stack carries nothing, and whatever is served at
  the endpoint answers. A borrowed model is declared by `url`, `model`,
  and `protocols`, which the builder writes to `endpoint.json`.

`source` is spelled the way the builder spells any other source (§6.3,
§6.5), and this proposal adds no scheme for it. Two examples of what
pinned means:

- A single file on Hugging Face, at a commit:
  `https://huggingface.co/Qwen/Qwen3-8B-GGUF/resolve/<commit>/Qwen3-8B-Q4_K_M.gguf`,
  with the file's SHA-256. Hugging Face already publishes a SHA-256 for
  every file it stores with Git LFS, which is how most weights are
  stored.
- A model in several files, such as an MLX model directory: the
  repository at a commit, with a digest over the whole directory, the
  way a builder pins any other source tree.

A branch or tag such as `main` or `latest` is not a revision, and a
name in a model runner's registry is pinned only once it is resolved
to a digest.

§6.4 gains three lines. A declaration does not say which model to use
for which task; that is the agent client's choice among the models it
is given. It does not say what hardware a model needs; `check` tests
for that on the host. And it does not say what else the agent client
may reach; that belongs to the `sandbox` field of Proposal 0001.

### 5. Deployment patterns

Carried and borrowed between them cover the places a model is served
today:

| Where the model is served | Form | Who starts and stops it |
|---|---|---|
| On the host, by the stack | carried | the stack ([Launch](#6-launch)) |
| On the host, by the consumer, such as a server they run for everything | borrowed, with a loopback `url` | the consumer |
| On a machine the consumer's organization runs: a GPU server on its network, inference in its own cloud account | borrowed | whoever operates it |
| At a hosted provider: the agent client's vendor, or one that serves open models | borrowed | the provider |

A stack running on another machine, such as a GPU workstation or a VM,
is the first row on that machine.

One stack can combine rows. A stack that carries a small model and
borrows a larger one from a team server gives its agent client both,
and the agent client decides which to use for what. A stack that must
only ever use the first row says so with a sandbox.

Borrowed models are also how an organization can hand its endpoints to
its stacks: a package holding only
`share/agent-models/<name>/endpoint.json` is installed like any other
(see [Bindings](#8-bindings)).

### 6. Launch

The stack owns the lifecycle of every server it starts. A launcher that
serves models SHOULD keep three properties:

1. **Nothing to set up.** Launching the stack is enough. It needs no
   server installed, started, or configured on the host beforehand, and
   no model downloaded.
2. **Nothing left behind.** Once the last agent client the stack
   started has exited, however it exited, every process the stack
   started for its models has exited too.
3. **Nothing shared beyond the stack.** A server's port, state, and
   caches are this stack's alone: no other stack, and nothing else the
   consumer runs, uses them. A built stack is never written to (§7), so
   they live in a state directory the launcher keeps for the stack on
   the host, the way staging lives outside the stack. A launcher never
   attaches to a server the stack did not start; a server the consumer
   already runs is a borrowed model, and is declared as one.

Two lifetimes keep them, and either conforms:

- **Per launch.** Each launch starts its own servers and stops them
  when its agent client exits. It is the simplest, and each launch pays
  to load each model and holds its own copy in memory.
- **Per stack.** The first launch starts the servers, concurrent
  launches of the same stack share them, and the last to exit stops
  them. Sharing saves memory and keeps caches warm, and because it
  never crosses stacks, "which model saw this?" keeps one answer.

Neither keeps a server running between launches that do not overlap. A
consumer who expects back-to-back work keeps one launch running.

A stack that carries several models starts a server for each, and holds
all of them in memory for as long as it runs. Starting a server only
when its model is first asked for would need something between the
agent client and the servers; see open question 2.

Keeping properties 2 and 3 takes four things that a server packaged
for everyday use usually does differently:

- **Stopping that survives a crash.** The server has to stop when the
  agent client crashes, and when whatever watches it is killed, so
  stopping it cannot depend on any one process exiting cleanly.
- **No fixed port.** Inference servers default to fixed ports. A fixed
  port collides with a second stack or with a server the consumer runs,
  and at worst connects the agent client to that server, which serves
  some other model.
- **State in the stack's state directory.** The server's home and data
  directories are redirected there. Left alone, Ollama, for one, writes
  models and a key pair under the consumer's home directory.
- **Weights from the build.** The server reads the weights from the
  dependency set, read-only, rather than downloading them into a cache
  on first run.

A prompt cache written to disk holds what the agent client was given,
in derived form, and outlives the session that produced it unless
something removes it. A launcher SHOULD NOT keep one past the server's
lifetime unless the consumer asks, and SHOULD keep any it does keep in
the stack's state directory, where it can be found and wiped.

**Serve.** §7 lists four steps. This proposal adds one between
resolving the agent and executing it, ahead of Proposal 0001's wrap
step when both apply. When the stack has models:

1. **Start or join the servers.** For each carried model with no
   server running for this stack, run `check` if there is one, choose
   a loopback address and a free port, run `serve` with them and a
   protocol the agent client speaks, and wait for its line. Under a
   per-stack lifetime, join a server that is already running. For each
   borrowed model, check that the endpoint answers and speaks a
   protocol the agent client speaks.
2. **Give the models to the agent client**, through the agent client's
   own configuration (an environment variable, a generated file in the
   per-launch staging directory), never through the consumer's (§9.1).
   Each model is given by name, and `defaultModel` as the one it starts
   with.
3. **Bind the lifetimes**, per launch or per stack, as above.

Agent clients differ in how they take several models. One that accepts
a list of providers gets one entry per model. One that takes a single
endpoint per protocol, as Claude Code takes one base URL, can reach
several models only if they answer at one address; open question 2
asks whether the launcher should arrange that.

A launcher that serves models also:

- SHOULD refuse to launch, naming the model and the reason, with the
  code when `check` or `serve` gave one, when a declared model cannot
  be served, its endpoint does not answer, or the agent client cannot
  be given it. It SHOULD NOT start the agent client against its own
  default model instead. A stack that declares a model and quietly runs
  without it looks like one stack and is another, which is the failure
  §6.2 exists to prevent.
- SHOULD announce borrowed models on every launch, naming each endpoint
  and saying whether it is a loopback address, as §9.3 asks for a
  borrowed agent client.
- SHOULD bind a carried model's server to loopback only.
- SHOULD make the stack's models inspectable before launch: each
  model's name and form, where a carried model's weights and server
  resolve, and a borrowed model's endpoint.

§7 also says the launcher hands over to the agent, gets out of the
way, and is not a supervisor. A model server is the first thing a
launch starts besides the agent client, and the first that has to be
stopped afterwards, so something has to watch. This proposal
recommends that the launcher still execute once, into a small wrapper
in the shape of Proposal 0001's sandbox:

```sh
<serve wrapper> -- [<sandbox> [options] --] <agent client> [args]
```

The wrapper starts or joins the servers (each under the stack's
sandbox when it has one; see [With a sandbox](#7-with-a-sandbox)),
sends their output to the stack's state directory rather than to the
terminal, runs what follows `--` with the terminal left to the agent
client, and returns its exit status. Under a per-launch lifetime each
wrapper stops its own servers; under a per-stack one, the last wrapper
to exit stops the shared ones. An implementation with a service manager
of its own can use that instead, as long as the three properties hold.

**Cost.** This part is non-normative. Starting a server is cheap.
Loading a model is not, and neither is the first request of a session:

| Cost | Paid | Size |
|---|---|---|
| Start the server process | each start | negligible |
| Load the weights | each start | grows with the model; a cold start reads the whole file, a warm one mostly hits the operating system's page cache |
| Process the agent client's system prompt and tool definitions | the first request of each session | thousands of tokens, which can take from seconds to a minute or more on consumer hardware |
| Hold the model in memory | each running server | the model's size, once per server |

A long interactive session absorbs these. Many short launches pay
them every time, a loop that runs the agent client once per task most
of all. A per-stack lifetime is for that case, and so is keeping one
launch running. The sizes are estimates until the rollout measures
them.

### 7. With a sandbox

Proposal 0001's wrap step resolves an effective sandbox configuration
from what the stack implies, including the agent client's own API and
authentication hosts. When the stack has models, the launcher adds
each model's address to what the agent client may reach: a carried
model's loopback address, and a borrowed model's host. When the stack
gives the agent client a default model, the agent client's vendor API
hosts are no longer implied. A sandbox that denies all other traffic
still lets the agent client reach the stack's models.

The servers run under the sandbox too. A model server is someone
else's software, and it can make network requests like any other
program. A sandbox with a strict network policy should hold for every
process the stack starts, not only the agent client.

Each server runs in its own instance of the stack's sandbox, under an
effective configuration derived for it: the stack's base configuration,
plus read access to its model directory and the dependency set, write
access to its state directory, listening on its loopback address, and
its accelerator, with no remote hosts. It gets its own instance rather
than sharing the agent client's for two reasons. Concurrent launches
can then share a server under a per-stack lifetime. And each process
gets only what it needs: the server no remote hosts, the agent client
no accelerator.

A stack that carries its models and a sandbox, and whose plugins name
no remote hosts, therefore runs with its models as the agent client's
only destinations, and with servers that can reach nothing at all.
That is the form need 5 asks for. This proposal decides which models
the agent client is given, and the sandbox decides that nothing else
goes anywhere.

Two things this depends on are untested, and the rollout has to
establish them. One is whether each sandbox in Proposal 0001 can give
a server its accelerator: the GPU through Seatbelt on macOS, and its
device files through bubblewrap or Landlock on Linux. The other is
whether each can allow a few loopback addresses and deny the rest.
Open questions 3 and 4 cover what happens if not.

### 8. Bindings

This subsection is non-normative. It follows §6.5: the same fields,
spelled the way each builder spells everything else.

As a Nix function argument, carried:

```nix
mkAgentStack {
  name = "caveman-stack";
  harness = claude-code;
  plugins = [ agent-plugin-caveman ];
  models = [ agent-model-qwen3-8b ];
}
```

As with a plugin, the model's own fields (its source and its server)
live in the model's package, and the stack expression only names it.
With one model, that model is also the default.

As an environment manifest, where the environment is the declaration
and installing a package that provides
`share/agent-models/<model-name>/` is how a model is added:

```toml
[install]
claude-code.pkg-path          = "claude-code"
agent-plugin-caveman.pkg-path = "agent-stacks/agent-plugin-caveman"
agent-model-qwen3-8b.pkg-path = "agent-stacks/agent-model-qwen3-8b"
```

A borrowed model is installed the same way, from a package holding only
its `endpoint.json`. That is how an organization can publish the
endpoints its stacks are meant to use.

As a command line, `agent-stacks launch claude` (§6.5) already takes
plugins from the active environment's `share/agent-plugins/`. It takes
models from `share/agent-models/` the same way, so it needs no new
flags, except one naming the default when the environment holds more
than one model.

All three put the choice on disk, where an author can distribute it
and a consumer can inspect it before running anything.

## Considered alternatives

**Borrowed endpoints only.** The quickest to build: the stack names a
URL and the consumer runs the server. Kept, as the borrowed form, which
is also how a stack reaches a model on another machine. On its own it
meets neither need 1 nor need 2: the model is not pinned or installed
with the rest, and the server has to be running before the stack
starts and keeps running after.

**Resolving the server from the weights' format.** A builder could read
the weights' format and pick its own default server for it, with the
declaration overriding that, the way a plugin's runtimes follow from
its shebangs. Rejected: a shebang names one runtime, but a format does
not name a server. A GGUF file runs on llama.cpp or on Ollama, which
differ in protocols, flags, and performance on the same machine, and a
choice that changes those should be written down where a consumer can
see it.

**The agent client's own local-model support.** Several agent clients
can already be pointed at a local server. A launcher uses those
mechanisms to give the agent client its models; the stack does not
expose them. A stack's models, like its sandbox in Proposal 0001,
should not change when its agent client does.

**A model as a plugin.** A model could be packaged as a plugin, with
its server as the plugin's runtime. Rejected: a plugin's runtimes exist
for that plugin's skills and MCP servers, and the model is for the
agent client itself, whether or not the stack has any plugins. A stack
with one model and no plugins is a reasonable thing to build. A model
that a plugin uses internally does stay with that plugin
([Non-goals](#non-goals)).

**A manifest for every model.** A file beside each model naming its
server, protocols, parameters, and requirements. Not adopted for
carried models, for the reason §11.1 gives for a stack manifest: the
links in the model directory already say which weights and which
server, `serve` holds the parameters, and `check` covers the
requirements, so a manifest would restate them. A borrowed model has
nothing to run and nothing to link, so for it a file is the only
representation there is, and `endpoint.json` is kept to the three
fields a launcher needs.

**Host requirements as data.** Three ways to fail early were compared:

- Recording requirements as data, such as memory in bytes or a class
  of accelerator, lets a launcher or a catalog decide before running
  anything. But the specification would have to define each field
  across hardware it does not want to name: unified memory against
  dedicated video memory, which accelerators belong to which class.
- Failing only when the model loads costs authors nothing, but the
  consumer waits through a load to learn that it was never going to
  work.
- An optional `check` gives the early failure wherever a model's author
  can write one, needs no vocabulary for hardware, and falls back to
  the second option when it is absent.

The third is adopted.

**A host-wide model service.** One long-running server per machine,
which is how several local model runners install by default. Rejected
for carried models: it has to be set up before the stack runs and
outlives it, which breaks properties 1 and 2, and its models are
shared across stacks, which breaks property 3. A per-stack lifetime
keeps most of what it saves. A consumer who runs one reaches it as a
borrowed model.

**A composed wire protocol.** §4 composes specifications by version.
The API between an agent client and its model has candidates, Open
Responses among them, but none that agent clients and servers can
both be relied on to speak today. Until one settles, protocols are
named (Appendix A) and matched at launch, and `serve` refuses a
protocol its server does not speak.

## Rollout

Each step is usable on its own, and each tells the next what to keep.

1. **Point and refuse.** A borrowed model on loopback, Claude Code, and
   llama-server: the agent client given the model through its own
   configuration, and the launch refused when the endpoint does not
   answer. This tests need 3 and the `endpoint.json` layout, and
   changes nothing a stack carries. llama-server first because it is
   one executable with no daemon, loads weights from a path it is
   given, runs on macOS and Linux, and serves both the Anthropic
   Messages API and the OpenAI chat completions API, so one server
   covers Claude Code and the agent clients that speak chat
   completions.
2. **Carry the model.** A model package with weights, and a `serve` and
   a `check` over llama-server, on a per-launch lifetime. Three tests
   matter. Copy the stack and its dependency set to a machine with
   networking off, and run it: that is the second use case. Kill the
   agent client abruptly, and check that nothing the stack started is
   still running: that is property 2. Declare a model too large for the
   host, and check that the launch fails early with `memory`. Measure
   the costs in [Launch](#6-launch) along the way.
3. **Enforce.** With Proposal 0001's sandbox around both the server and
   the agent client: the agent client reaches its model, a request to
   its vendor's API fails, the server reaches nothing, and the server
   still gets its accelerator.
4. **A second server.** Ollama or oMLX, with an agent client that
   speaks chat completions. This tests naming the server in the
   declaration, whether either server will serve read-only weights from
   the dependency set with its state redirected, and whether matching
   protocols at launch is the right boundary.
5. **Several models.** A carried model and one borrowed from another
   machine, with an agent client that takes several providers. This
   tests giving several models to one agent client, and announcing a
   remote endpoint.
6. **A per-stack lifetime.** Concurrent launches of one stack sharing
   servers, stopped by the last to exit, with the same abrupt-exit
   check as step 2.
7. **Into the specification.** Fold what survives into a numbered
   version, with the protocol names moved to a general section and the
   open questions below answered or restated.

## Open questions

1. **Context length.** The context length is fixed in `serve` when the
   stack is built. How does the agent client learn it, so that it
   compacts before it overflows? `serve` could report it on its ready
   line, but a launcher would still need a way to tell each agent
   client.
2. **Several models at one address.** An agent client that takes one
   endpoint per protocol, as Claude Code does, can reach several models
   only if they answer at one address. Should a launcher put them
   behind one, sending each request to the server for the model it
   names? The agent client would still choose the model, so this is not
   routing, but it is one more process a launch starts. It would also
   let a server start only when its model is first asked for, rather
   than at launch.
3. **Servers outside the sandbox.** If a sandbox cannot give a server
   its accelerator, should a stack be able to run its servers outside
   the sandbox, with only the agent client inside, and how would a
   consumer see that it does?
4. **Loopback addresses.** Can both sandboxes in Proposal 0001 allow
   the stack's model addresses and deny the rest of loopback? If not,
   the agent client can reach anything else listening on loopback along
   with its models.
5. **Credentials for a borrowed endpoint.** `endpoint.json` holds none,
   but an endpoint on another machine usually needs one. Secrets are
   open in §11.3. Until they are settled, how does a launcher give a
   credential to the agent client without putting it in the stack?

## Appendix A: Protocol names

This appendix is non-normative, and grows with each agent client and
server tested. Protocol names are not specific to models: if this
proposal is accepted, they move to a general section of the
specification. `serve --protocol`, `check --protocol`, and the
`protocols` field of `endpoint.json` all use them, and each name's
path is appended to an endpoint's `url`.

| Name | API |
|---|---|
| `anthropic-messages` | Anthropic Messages API, `POST /v1/messages` |
| `openai-chat-completions` | OpenAI Chat Completions API, `POST /v1/chat/completions` |
| `open-responses` | [Open Responses](https://openresponses.org/), the open specification of OpenAI's Responses API, `POST /v1/responses` |

Claude Code speaks `anthropic-messages`. llama-server serves
`openai-chat-completions`, and `anthropic-messages`
[since January 2026](https://huggingface.co/blog/ggml-org/anthropic-messages-api-in-llamacpp).
