# Proposal 0002: Running a stack's agent client against a local model

- Status: Draft, for discussion. Experimental; like everything before
  1.0.0, anything here may change or be withdrawn.
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.3, and the sections it
  would touch: §5.1, §6.1, §6.4, §7, §8, §9, §10. Composes with
  [Proposal 0001](0001-host-sandbox.md)
  ([#4](https://github.com/agent-stacks/agent-stacks-spec/pull/4)).
- Scope: one model per stack, served on the host where the stack runs,
  designed for two use cases. Everything else is listed under
  [Non-goals](#non-goals).

## Summary

A stack says which agent client runs and what it carries. It says
nothing about the model that agent sends its prompts to, so in
practice the agent sends them to its vendor's hosted API. This
proposal lets a stack name that model instead, and carry it.

Four parts, each small enough to change later:

1. **Layout.** A stack MAY carry one model at
   `<stack root>/share/agent-models/<model-name>/`: its weights, the
   server that runs them, and a `bin/serve` executable that joins the
   two.
2. **Build.** The builder resolves the server the way §8 resolves a
   plugin's runtimes: from the weights' format, with a default per
   format that a declaration can override, failing the build when
   there is none.
3. **Declaration.** The vocabulary in §6.1 gains an optional `model`
   field. The carried and borrowed distinction of §6.2 applies: a stack
   carries its model, or names an endpoint where one is already served.
4. **Launch.** The stack owns the server's whole lifecycle. Launching
   the stack starts the server on a loopback port and points every
   model call the agent client makes at it. Once the last agent client
   the stack started has exited, nothing the stack started for the
   model is still running. When the model cannot be served, the
   launcher refuses to launch rather than fall back to a hosted model.

Together they give three levels of assurance, and a consumer can tell
which one they have:

| Form | Prompts go to | The model is | Checked by |
|---|---|---|---|
| Borrowed | the endpoint the stack names | whatever is served there | the endpoint, announced at every launch |
| Carried | a loopback port on this host | pinned by digest, in the stack | inspecting the stack before it runs |
| Carried, with a sandbox | that port, and nowhere else | pinned by digest, in the stack | the stack and the effective sandbox configuration |

No new model format and no new wire protocol are proposed.

## Motivation

### Two use cases

This proposal is designed against two use cases, and should be judged
by how well it serves them.

**Data that does not leave the machine.** A person or a team wants an
agent to work on data they will not send to a third party: medical or
financial records, a private codebase, personal notes and measurements.
They will accept a less capable model. They will not accept one that is
sometimes hosted, because what they chose was the guarantee, not the
model.

**Compliance where the data lives.** An organization holds data under
rules that restrict where it may be processed, and wants an agent on
hardware already inside those rules, possibly with no network at all.
It needs to install the whole thing offline, and to show someone else,
before it runs, what it is and where it can send data.

### What they need

1. **No silent fallback.** When the local model is unavailable, the
   agent does not run. A fallback to a hosted model is the one failure
   neither use case can absorb.
2. **Every model call goes to the local model.** Not only the
   conversation. An agent client may use more than one model; Claude
   Code, for one, uses a smaller model for background work, and
   pointing only the main one sends the rest to the hosted API.
3. **The model travels with the stack.** Pinned by digest, installed
   with everything else, never downloaded on first run, so that the
   stack runs on a machine with no network. This is §5.1(6) applied to
   one more thing.
4. **Inspectable before it runs.** Which model, served where, and every
   other place the stack can send data.
5. **Enforced, not only configured.** Configuration decides where
   inference goes. Only a sandbox decides that nothing else goes
   anywhere.

The first use case needs 1 and 2. The second needs all five.

### What a local model needs

A plugin needs runtimes, and §8 carries them so that the stack does not
depend on what the consumer has installed. A local model needs the
same, and one thing more:

| Layer | Examples | Plugin counterpart | Carried? |
|---|---|---|---|
| Weights | a GGUF file, an MLX model directory | the plugin's content | Yes, pinned by digest like any other source |
| Server | llama.cpp, Ollama, oMLX, vLLM | a runtime such as `python3` or `node` | Yes, resolved by the builder as §8 resolves runtimes |
| Host | an accelerator, its driver, memory | the kernel an interpreter runs on | No |

The first two are what this proposal adds to a stack. The third is what
a stack built to it still asks of the machine, and
[What the stack cannot carry](#3-what-the-stack-cannot-carry) says how
a launcher handles it.

### What it optimizes for

- **Start small.** One model, one host, one stack. Each is the
  smallest version of its axis, and each can widen later without
  undoing what is here.
- **The framework, not today's models.** The proposal names no model,
  parameter count, quantization, context window, or hardware. Those
  change every few months. What a stack has to say about a model does
  not: where its weights are, what serves them, where the agent sends
  its prompts, and what happens when that fails. A stack built to this
  proposal behaves the same whether the model it carries is good
  enough for its task this year or only next.
- **Two use cases, served well.** Cost savings, and workflows that mix
  local and hosted models, get whatever falls out of the design. They
  are not what it was tested against, because both depend on how
  capable today's local models are, which is what the previous point
  declines to design around.

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
- **Routing between models.** Local first with escalation to a hosted
  model, a hosted model that plans and a local one that executes, a
  different model per role. Each is a real workflow, and each puts a
  router's decision between the consumer and the question of where
  their data went. They can already be expressed by nesting (§5.4): a
  stack per model, and an outer stack whose agent client is the loop or
  session manager that moves work between them. Each inner stack keeps
  a boundary that can be stated in one word.
- **Serving a model on another machine.** A carried model is served on
  the host the stack runs on. A model on a shared server is reachable
  as a borrowed endpoint; starting and stopping it there is out of
  scope.
- **One server shared between stacks.** A server serves the stack that
  started it. Concurrent launches of that stack may share it
  ([Launch](#5-launch)); another stack may not. A server the consumer
  runs for everything is reached as a borrowed endpoint.
- **Models a plugin uses.** An embedding model behind a plugin's MCP
  server is that plugin's runtime, carried as §8 carries any other.
  This proposal is about the model the agent client itself talks to.
- **How weights are distributed.** Registries, mirrors and size limits
  are the builder's concern. The proposal asks only that weights be
  pinned and carried like any other source.

## Proposal

### 1. Layout

1. A stack MAY carry a model at
   `<stack root>/share/agent-models/<model-name>/`.
2. A stack SHOULD carry at most one. A stack names one agent client
   (§5.4); two models would ask the launcher to choose, and would give
   "which model saw this?" two answers.
3. The directory SHOULD hold the model's weights and a `bin/` holding
   an executable named `serve` beside a link to the server it runs. The
   launcher reads neither the weights nor the server. It runs `serve`.
4. The weights and the server SHOULD be links into the stack's
   dependency set, as a plugin's runtimes are (§8.3). In a
   content-addressed store, the path a link resolves to names the
   digest of what it points at, which is how a consumer can see which
   model a stack carries without a manifest.

```text
caveman-stack/
├── bin/
│   └── caveman-stack
└── share/
    ├── agent-plugins/
    │   └── caveman/...
    └── agent-models/
        └── qwen3-8b/
            ├── bin/
            │   ├── serve              the serve contract
            │   └── llama-server       link to the server it runs
            └── qwen3-8b-q4_k_m.gguf   link to the weights
```

This is the move §5.1 makes for plugins and Proposal 0001 makes for
sandboxes: a directory under `share/` per thing, discoverable by
listing it. A stack with no `agent-models/` directory runs exactly as
it does today.

`serve` is the whole interface between a launcher and a model. The
launcher runs it as:

```sh
<model dir>/bin/serve --listen 127.0.0.1:<port> --protocol <name>
```

and `serve`:

- serves the model at that address, over that protocol, in the
  foreground, until it receives `SIGTERM`;
- exits non-zero, with the reason on standard error, when it cannot:
  a protocol its server does not speak, weights that will not load, an
  address it cannot bind, a host that lacks what the server needs;
- writes one line to standard output once the model is loaded and
  ready to answer, and nothing to standard output before that.

Protocol names are listed in Appendix A. Everything else (which
server, its flags, the context length it allocates) is settled when
the stack is built. A launcher needs to know nothing about the server,
so supporting a new one is work for the builder and none for the
launcher.

### 2. Build

A model's weights are a source like any other, and §10.1 applies: a
builder SHOULD pin them by digest and fetch them when the stack is
built, never when it is launched. A stack that downloads its model the
first time it runs is not transportable (§5.1(6)), cannot run where
there is no network, and leaves the download behind when it is gone.

The server is resolved the way §8 resolves a plugin's runtimes, and
for the same reason: so that the stack does not depend on what the
consumer has installed.

1. A builder SHOULD determine the weights' format from the weights
   where the format says what it is, as a GGUF file does in its first
   four bytes, and from the declaration where it does not. This is the
   rule Appendix B.2 gives for runtimes: read what the format fixes,
   and have the rest declared.
2. A builder SHOULD resolve a server for that format from its own
   defaults, unless the declaration names one. A shebang names exactly
   one runtime, but a format admits several servers: a GGUF file runs
   on llama.cpp or Ollama, an MLX model on oMLX or mlx-lm. That is why
   a declaration can choose, as a plugin's `dependencies` can (§6.3).
3. A builder that cannot resolve a server for the format SHOULD fail
   the build, naming the model, its format, and how a declaration names
   a server, as §8.4 asks for a runtime.
4. A builder SHOULD generate `serve`, and link the server and the
   weights into the model directory, as it links a plugin's runtimes
   (§8.3). `serve` is to a model what a rewritten shebang is to a
   script: the builder's glue between content and the runtime it
   needs.

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

When `serve` fails because of the host, the launcher SHOULD say so in
those terms, naming what the host lacks where the server reports it,
rather than leave the consumer to read a server's log.

The proposal does not say how much memory or which accelerator a model
needs. Those are facts about one model on one machine, and how a stack
could record them without the specification naming hardware is an
open question.

### 4. Declaration

§6.1 gains one optional field:

- `model` — the model the agent client sends its prompts to, and how
  it is obtained.

§6.2 applies, and a declaration SHOULD say which form it takes rather
than leave that to which fields are present:

- **Carried.** The model is part of the stack's dependency set, and the
  stack runs the same model on every machine it reaches, including one
  with no network. A carried model is declared by:
  - `source` — where the weights come from, pinned by revision and
    content digest.
  - `format` — the weights' format, only where the weights do not say
    it themselves.
  - `server` — the package to serve them with, named as the builder
    names packages, only where the builder's default for the format is
    not wanted.
- **Borrowed.** The stack carries nothing, and whatever is served at
  the endpoint answers. A borrowed model is declared by:
  - `endpoint` — the address the model is served at.
  - `name` — the model to request there.

  It is recorded where a borrowed agent client is: in the launcher the
  builder produces.

§6.4 gains two lines. A declaration does not say which model to use
for which task, or which model to fall back to: a stack has one, and
when it is unavailable the stack does not run. Nor does a declaration
say what hardware a model needs; that is a fact about the model and
the machine, not about the stack.

### 5. Launch

The stack owns the server's whole lifecycle. A launcher that serves a
model SHOULD keep three properties:

1. **Nothing to set up.** Launching the stack is enough. It needs no
   server installed, started, or configured on the host beforehand, and
   no model downloaded.
2. **Nothing left behind.** Once the last agent client the stack
   started has exited, however it exited, every process the stack
   started for the model has exited too.
3. **Nothing shared beyond the stack.** The server's port, state, and
   caches belong to the stack. They live outside it, as staging does
   (§7), and apart from anything the consumer uses. A launcher never
   attaches to a server the stack did not start; a server the consumer
   already runs is a borrowed model, and is declared as one.

Two lifetimes keep them, and either conforms:

- **Per launch.** Each launch starts its own server and stops it when
  its agent client exits. It is the simplest, and each launch pays to
  load the model and holds its own copy in memory.
- **Per stack.** The first launch starts the server, concurrent
  launches of the same stack share it, and the last to exit stops it.
  The services of a Flox environment, which stop when its last
  activation exits, already behave this way. Sharing saves memory and
  keeps caches warm, and because it never crosses stacks, "which model
  saw this?" keeps one answer.

Neither keeps a server warm between launches that do not overlap; see
open question 8.

Keeping properties 2 and 3 takes four things that a server packaged
for everyday use usually does differently:

- **Stopping that survives a crash.** The server has to stop when the
  agent client crashes, and when whatever watches it is killed, so
  stopping it cannot depend on any one process exiting cleanly.
- **No fixed port.** Inference servers default to fixed ports. A fixed
  port collides with a second stack or with a server the consumer runs,
  and at worst connects the agent client to that server, which serves
  some other model.
- **State inside the stack.** The server's home and data directories
  are redirected into the stack's state. Left alone, Ollama, for one,
  writes models and a key pair under the consumer's home directory.
- **Weights from the build.** The server reads the weights from the
  dependency set, read-only, rather than downloading them into a cache
  on first run.

A prompt cache written to disk holds what the agent client was given,
in derived form, and outlives the session that produced it unless
something removes it. A launcher SHOULD NOT keep one past
the server's lifetime unless the consumer asks, and SHOULD keep any it
does keep in the stack's state, where it can be found and wiped.

**Serve.** §7 lists four steps. This proposal adds one between
resolving the agent and executing it, ahead of Proposal 0001's wrap
step when both apply. When the stack has a model:

1. **Start or join the server.** With no server running for this
   stack, choose a loopback address and a free port, run `serve` with
   them and the protocol the agent client speaks, and wait for its
   ready line. With one already running, under a per-stack lifetime,
   use it. For a borrowed model, check instead that the endpoint
   answers.
2. **Point the agent client at it**, through the agent client's own
   configuration (an environment variable, a generated file in the
   per-launch staging directory), never through the consumer's (§9.1).
   Every model the agent client uses SHOULD be pointed at the stack's
   model, not only the main one. Where the agent client offers a switch
   for its other calls to its vendor, such as telemetry or update
   checks, the launcher SHOULD turn them off.
3. **Bind the lifetimes**, per launch or per stack, as above.

A launcher that serves a model also:

- SHOULD refuse to launch, with a message naming the reason, when the
  model cannot be served, the endpoint does not answer, or the agent
  client cannot be pointed at it. It SHOULD NOT fall back to the agent
  client's default model. A stack that looks local and is not is the
  failure this proposal exists to prevent, as a stack that looks
  reproducible and is not is the one §6.2 exists to prevent.
- SHOULD announce a borrowed model on every launch, naming the
  endpoint and saying whether it is a loopback address, as §9.3 asks
  for a borrowed agent client. A launcher cannot know who operates a
  borrowed endpoint. It can say where it is sending.
- SHOULD bind a carried model's server to loopback only.
- SHOULD make inspectable before launch the model, where its weights
  resolve, the endpoint, and every other host the stack can reach: the
  remote MCP servers its plugins name, and any vendor host the agent
  client could not be told to leave alone. A local model does not make
  a stack local if a plugin sends what the agent gives it to a hosted
  service, and the consumer should see both in the same place.

This proposal adds no consumer override that sends a stack's
inference to a hosted model. A consumer who wants the same agent
against a hosted model runs a stack without a model. That is two
stacks, as §5.4 already says.

§7 also says the launcher hands over to the agent, gets out of the
way, and is not a supervisor. A model server is the first thing a
launch starts besides the agent client, and the first that has to be
stopped afterwards, so something has to watch. This proposal
recommends that the launcher still execute once, into a small wrapper
in the shape of Proposal 0001's sandbox:

```sh
<serve wrapper> -- [<sandbox> [options] --] <agent client> [args]
```

The wrapper starts or joins the server, sends the server's output to
the stack's state rather than to the terminal, runs what follows `--`
with the terminal left to the agent client, and returns its exit
status. Under a per-launch lifetime each wrapper stops its own server;
under a per-stack one, the last wrapper to exit stops the shared one.
An implementation with a service manager of its own can use that
instead, as long as the three properties hold.

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
of all, and they are what a per-stack lifetime and open question 8 are
for. The sizes are estimates until the rollout measures them.

### 6. With a sandbox

Proposal 0001's wrap step resolves an effective sandbox configuration
from what the stack implies, including the agent client's own API and
authentication hosts. When the stack carries a model, this proposal
changes that one input: the agent client's API hosts are replaced by
the model's loopback endpoint.

The server runs outside the sandbox, and the agent client inside it.
The server is pinned software that needs the machine's accelerator;
the agent client is the thing whose reach the consumer wants to bound.
A stack that carries a model and a sandbox, and whose plugins name no
remote hosts, therefore runs with one allowed destination: its model.

That is the form need 5 asks for, and the reason the two proposals are
written to compose rather than overlap. This one decides where
inference goes. The sandbox decides that nothing else goes anywhere.

### 7. Bindings

This subsection is non-normative. It follows §6.5: the same field,
spelled the way each builder spells everything else.

As a Nix function argument, carried:

```nix
mkAgentStack {
  name = "caveman-stack";
  harness = claude-code;
  plugins = [ agent-plugin-caveman ];
  model = agent-model-qwen3-8b;
}
```

As with a plugin, the model's own fields (its source, and a server
where the default is not wanted) live in the model's package, and the
stack expression only names it.

As an environment manifest, where the environment is the declaration
and installing a package that provides
`share/agent-models/<model-name>/` is how the field is set:

```toml
[install]
claude-code.pkg-path          = "claude-code"
agent-plugin-caveman.pkg-path = "agent-stacks/agent-plugin-caveman"
agent-model-qwen3-8b.pkg-path = "agent-stacks/agent-model-qwen3-8b"
```

As a command line, borrowed, for a stack assembled for one launch:

```sh
agent-stacks launch --model-endpoint http://127.0.0.1:11434 \
  --model qwen3:8b claude
```

The first two carry the model and put the choice on disk, where an
author can distribute it and a consumer can inspect it before running
anything. The third borrows a model the consumer already serves.

## Considered alternatives

**Borrowed endpoints only.** The quickest to build: the stack names a
URL and the consumer runs the server. Kept, as the borrowed form. On
its own it meets neither need 3 nor need 4: the model is not pinned,
not part of the unit, not installed offline with the rest, and nothing
a consumer can inspect says what will answer.

**Naming the server every time.** A declaration could be required to
name the server as well as the weights. Rejected for the reason §6.3
lets a plugin's runtimes resolve from defaults: most weights have an
obvious server, and a declaration should say what is unusual about a
model rather than restate what is not.

**The agent client's own local-model support.** Several agent clients
can already be pointed at a local server. A launcher uses those
mechanisms to point the agent; the stack does not expose them. Each
agent client's switch covers a different subset of its model calls,
which is what need 2 is about, and a stack's model, like its sandbox
in Proposal 0001, should not change when its agent client does.

**A model manifest.** A file beside the weights naming the server, the
protocols it speaks, and its parameters. Deferred, for the reason
§11.1 gives for a stack manifest: `serve` answers the launcher's
questions by being run, and a format specified before anything needs
it would be specified from imagination.

**A host-wide model service.** One long-running server per machine,
which is how several local model runners install by default. Rejected
for carried models: it has to be set up before the stack runs and
outlives it, which breaks properties 1 and 2, and its models are
shared across stacks, which breaks property 3. A per-stack lifetime
keeps most of what it saves. A consumer who runs one reaches it as a
borrowed endpoint.

**A composed wire protocol.** §4 composes specifications by version.
The API between an agent client and its model has candidates, Open
Responses among them, but none that agent clients and servers can
both be relied on to speak today. Until one settles, protocols are
named (Appendix A) and matched at launch, and `serve` refuses a
protocol its server does not speak.

## Rollout

Each step is usable on its own, and each tells the next what to keep.

1. **Point and refuse.** A borrowed model, Claude Code, and
   llama-server on loopback: every model role pointed, the agent
   client's other vendor calls switched off, and the launch refused
   when the endpoint does not answer. This tests needs 1 and 2, and
   changes nothing a stack carries. llama-server first because it is
   one executable with no daemon, loads weights from a path it is
   given, runs on macOS and Linux, and serves both the Anthropic
   Messages API and the OpenAI chat completions API, so one server
   covers Claude Code and the agent clients that speak chat
   completions.
2. **Carry the model.** A model package with weights and a `serve`
   over llama-server, on a per-launch lifetime. Two tests matter. Copy
   the stack and its dependency set to a machine with networking off,
   and run it: that is the second use case. Then kill the agent client
   abruptly, and check that nothing the stack started is still running:
   that is property 2. Measure the costs in [Launch](#5-launch) along
   the way.
3. **Enforce.** With Proposal 0001's sandbox, check that the agent
   client reaches its model and that a request to its vendor's API
   fails.
4. **A second server.** Ollama or oMLX, with an agent client that
   speaks chat completions. This tests the builder's server defaults
   and overrides, whether either server will serve read-only weights
   from the dependency set with its state redirected, and whether
   matching protocols at launch is the right boundary.
5. **A per-stack lifetime.** Concurrent launches of one stack sharing
   a server, stopped by the last to exit, with the same abrupt-exit
   check as step 2.
6. **Into the specification.** Fold what survives into a numbered
   version, with the open questions below answered or restated.

## Open questions

1. **Serving parameters.** Context length, offload, and concurrency
   are set inside `serve` when the stack is built. Should a consumer be
   able to change them at launch, and how, without the stack restating
   a server's flags? And how does the agent client learn the context
   length it has to fit in?
2. **Agent clients that need their vendor to start.** An agent client
   that signs in, checks an entitlement, or updates before its first
   prompt cannot run on a machine with no network. Should a builder
   refuse to pair such an agent client with a model, or is refusing at
   launch enough?
3. **Local-only stacks.** Should a stack be able to say that nothing
   it runs may leave the host, so that a builder refuses a plugin that
   names a remote MCP server and a launcher refuses to run without a
   sandbox? It is the strongest form of need 5, and more policy than
   this specification has held so far.
4. **One loopback port.** Can both sandboxes in Proposal 0001 allow
   one loopback port and deny the rest of loopback? If not, "nowhere
   else" in the Summary holds for remote hosts only.
5. **Confining the server.** On a machine that is not air-gapped, the
   inference server runs with the consumer's network access. Pinned is
   not the same as confined. Should the server run under the stack's
   sandbox, under one of its own, or under neither?
6. **Where protocol names live.** In this specification, as Appendix A
   begins to, or in a composed specification once one settles?
7. **What a model asks of the host.** Should a model record what it
   needs from the machine, such as memory or a class of accelerator, so
   that a launcher can refuse before loading it? And can that be said
   without the specification naming hardware?
8. **A server kept warm.** Neither lifetime helps launches that do not
   overlap. Keeping the server for a short while after the last agent
   client exits would, at the cost of property 2 holding only after
   that delay. Is that a trade a consumer should be able to make?

## Appendix A: Protocol names

This appendix is non-normative, and grows with each agent client and
server tested.

| Name | API |
|---|---|
| `anthropic-messages` | Anthropic Messages API, `POST /v1/messages` |
| `openai-chat-completions` | OpenAI Chat Completions API, `POST /v1/chat/completions` |
| `open-responses` | [Open Responses](https://openresponses.org/), the open specification of OpenAI's Responses API |

Claude Code speaks `anthropic-messages`. llama-server serves
`openai-chat-completions`, and `anthropic-messages`
[since January 2026](https://huggingface.co/blog/ggml-org/anthropic-messages-api-in-llamacpp).
