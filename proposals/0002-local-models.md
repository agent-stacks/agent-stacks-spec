# Proposal 0002: Local models

- Status: Draft
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.3 and related sections
  (§5.1, §6.1, §7, §8, §9, §10).
- Scope: A stack can have any number of models. Some models are hosted
  by the stack itself and others are hosted elsewhere and referenced by
  the stack.

## Summary

Today, a stack is limited to whatever models are provided by an agent
client. This proposal makes models a first-class feature of a stack,
either hosted by the stack itself or hosted elsewhere and referenced by
the stack.

Defining a model in a stack has four unique considerations:

1. **Layout.** A stack MAY define models under
   `<stack root>/share/agent-models/`, one directory per model. A model
   hosted by the stack has its model files in that directory. A
   model hosted elsewhere is represented as an `endpoint.json` file
   saying where the model is hosted.
2. **Build.** For models hosted by the stack, the model file is fetched
   when the stack is built, and digest-pinned for reproducibility. The
   builder resolves the server, and links it similar to how §8 resolves
   a plugin's runtimes. The build SHOULD fail if either of these steps
   fails.
3. **Declaration.** The vocabulary in §6.1 defines `models`, either
   self-contained or externally-hosted per §6.2, and `defaultModel`,
   the model used by the agent client.
4. **Launch.** The stack owns the lifecycle of every inference server
   it starts. When you launch a stack, each inference server starts on
   a loopback port, which gives the agent client access to the models.
   These servers stop after the last agent client has exited. The
   launch will fail if a declared model can't be served.

Underneath all four, a model is reached through a model gateway that
speaks one of a few named protocols.
[§1](#1-model-gateways-and-protocols) names them and says what a
gateway does for a launcher to use it.

## Motivation

The choice of model has perhaps the biggest impact on how an agent
stack behaves. By making models part of the definition of a stack, we
can make stacks more self-contained and reproducible.

### Use cases

This proposal is informed by two real-world use cases:

**Data privacy.** A person or a team wants an agent to have access to
data without sending it to a third party. This could be medical or
financial records, a private codebase, or company IP.

**Compliance.** An organization may be required to comply with
regulations that restrict where data may be processed. If so, it may
need the ability to host a model on its own hardware, possibly with no
network access at all (e.g. in air-gapped environments).

### Architectural implications

1. **The model must be specified by the stack.** The agent stack must
   be explicit about what model is used. If the model is hosted by the
   stack itself, it must be pinned by digest and downloaded at build
   time, so that it can be launched on a machine with no network access
   (§5.1). If the model is hosted elsewhere, the stack must be explicit
   about where that model is hosted. This allows a third party to
   understand how the models in a stack are configured prior to
   launching the stack.
2. **The stack manages the lifecycle of inference servers.** If a model
   is hosted by the stack itself, there should be no external
   dependencies outside of the stack at runtime, other than hardware
   and drivers. The inference servers are started by the stack, and
   stopped by the stack.
3. **Declared models must be available.** If a model can't be served
   (in the self-contained case) or accessed (in the externally-hosted
   case), the stack will fail to launch.
4. **Models are not responsible for enforcing the behavior of the agent
   client.** If a stack declares a local model, and the agent client
   uses a cloud model instead, the only way to prevent this behavior is
   by restricting the agent client's network access. Notably, this is
   the only way to guarantee that data is not being leaked to a third
   party.

### Externally-hosted models

Open models are increasingly run somewhere other than the user's own
machine:

1. A virtual machine running in the cloud.
2. A GPU server in an organization's on-prem datacenter.
3. A third-party provider that hosts open models (e.g. OpenRouter).

An agent stack should be able to work with these externally-hosted
models too. These models are represented in the stack as an
`endpoint.json` file saying where the model is hosted. See
[Deployment patterns](#5-deployment-patterns) for more.

### Self-contained models

A self-contained model consists of:

1. The model files (e.g. GGUF, MLX).
2. The inference server (e.g. llama.cpp, Ollama, oMLX, vLLM) and its
   dependencies (e.g. CUDA).
3. A `bin/serve` executable that starts the server.

NOTE: A self-contained model MAY have specific hardware requirements
that limit what machines it can run on.

## Out of scope

- **Model routing.** A stack can give its agent client several models.
  However, a stack only ever defines a single default; automatically
  routing a request to the best model is not supported. A router
  that does so is a model gateway like any other (§1), and its rules
  are its own configuration.
- **Egress enforcement.** This proposal does nothing to limit an agent
  client's network access.
- **Management of externally-hosted models.** Externally-hosted models
  may be accessed by the agent client in an agent stack, but an agent
  stack does not otherwise manage those models or the model gateways
  that serve them.
- **Model distribution.** The only requirement is that self-contained
  models have pinnable model files. Model registries and download
  processes are out of scope.
- **Context management.** How an agent client manages its context is
  out of scope for this proposal.

## Terms

This proposal uses the terms of §3, and these:

- **Model** — what an agent client selects: an ID that a model gateway
  lists, with a context window and the protocols it is reached by. An
  ID MAY be an alias the gateway resolves to another model; a stack
  and its launcher treat it as opaque.
- **Model files** — what is on disk for a model that runs locally, in
  a format such as GGUF or MLX. An externally-hosted model has no
  model files.
- **Weights** — the tensor files among a model's files. The word is
  used only where their size, quantization, or digest matters.
- **Model gateway** — whatever serves models at an endpoint. Launch
  and the agent client only ever talk to a model gateway, and cannot
  tell its two kinds apart:
  - **Inference server** — a model gateway that runs models from model
    files: llama.cpp's `llama-server`, oMLX, Ollama, vLLM.
  - **Router** — a model gateway that forwards requests to other model
    gateways, and may translate between protocols or choose a backend
    per request. It runs no model itself: LiteLLM, claude-code-router.
- **Endpoint** — the base URL a model gateway is reached at, and the
  protocols it speaks there.
- **Protocol** — the API an endpoint speaks, by one of the names in
  [§1](#1-model-gateways-and-protocols).

"LLM gateway" is avoided on purpose. In the industry it names a proxy
in front of cloud providers that handles keys, budgets, and logging,
which is a narrower thing than a model gateway.

## Proposal

### 1. Model gateways and protocols

#### Protocol names

A stack, an endpoint, and an agent client name protocols, never
products. These are the names:

| Name | API |
| --- | --- |
| `anthropic-messages` | Anthropic Messages API, `POST /v1/messages` |
| `openai-chat-completions` | OpenAI Chat Completions API, `POST /v1/chat/completions` |
| `open-responses` | [Open Responses](https://openresponses.org/), the open specification of OpenAI's Responses API, `POST /v1/responses` |

An endpoint's URL includes the version prefix (e.g. `/v1`), and the
rest of the protocol's path is appended to it. These names are used by
`serve --protocol`, `check --protocol`, and the `protocols` field of
`endpoint.json`. The list grows with each agent client and model
gateway tested; a name that is not in it SHOULD NOT be used until it
is added.

No neutral standard covers all three. Model gateways copy vendor APIs,
and "compatible" means "mostly": tool calls, streaming events, and
reasoning output are where they drift. Only `open-responses` has an
open specification with acceptance tests.

#### What a model gateway does for launch

For a launcher to use it, a model gateway SHOULD:

1. **Serve every model at one endpoint.** One base URL for all the
   models it offers, whichever protocol a request uses.
2. **List every model at `GET /v1/models`**, in the OpenAI list shape
   (`{"object": "list", "data": [{"id": …}, …]}`), including models it
   has not loaded yet. An entry's `id` is the name a request uses.
3. **Load a model when a request names it.** A request whose `model`
   is a listed ID is served without a separate load call. The gateway
   MAY unload other models to make room.
4. **Speak at least one protocol** from the table above, at that
   protocol's path under the endpoint.

This is what lets one model gateway serve every model a stack has, and
launch hand the agent client one URL. Nothing else is assumed. How a
gateway reports a model's context window, its health, or what it has
loaded differs between gateways and is not part of the contract (see
[Open questions](#open-questions)). Appendix A records three inference
servers that meet it.

A router meets the contract the same way. Its routing rules are its
own configuration, as an inference server's flags are its own.

#### Matching protocols

An endpoint speaks one or more protocols, and so does an agent client.
Launch matches them; the agent client never works out the protocol
itself.

1. Launch SHOULD take the protocols both sides speak, and prefer the
   agent client's native one.
2. When there are none, launch SHOULD refuse to launch, naming the
   agent client, the endpoint, and what each speaks. It MAY suggest a
   router in front of the endpoint.
3. Launch then configures the agent client in the agent client's own
   terms (an environment variable, a flag, a staged file), which
   already imply the protocol.

What each agent client speaks is a fact about the agent client, and
lives in the launcher beside the rest of its per-agent code.

### 2. Layout

1. A stack MAY store models at
   `<stack root>/share/agent-models/<model-name>/`, one directory per
   model. Model names SHOULD be unique within a stack.
2. A self-contained model's directory SHOULD contain the model files
   and a `bin/` folder containing an executable named `serve` and a
   link to the server it runs. The `bin/` folder MAY also contain an
   executable named `check`.
3. An externally-hosted model's directory SHOULD contain one file,
   `endpoint.json`.
4. The model files and the server SHOULD be links into the stack's
   dependency set, similar to how plugin runtimes work (§8.3).
5. `<model-name>` is how the stack refers to the model.

```text
caveman-stack/
├── bin/
│   └── caveman-stack
└── share/
    ├── agent-plugins/
    │   └── caveman/...
    └── agent-models/
        ├── qwen3-8b/                  self-contained
        │   ├── bin/
        │   │   ├── serve              the serve contract
        │   │   ├── check              optional host check
        │   │   └── llama-server       link to the server it runs
        │   └── qwen3-8b-q4_k_m.gguf   link to the model file
        └── team-coder/                externally-hosted
            └── endpoint.json
```

The `agent-models/` directory shows you every model in the stack. If a
stack does not have an `agent-models/` directory, it runs exactly the
same as it does today.

#### `serve`

`serve` is the interface between a launcher and a self-contained model.
The launcher runs it as:

```sh
<model dir>/bin/serve --listen 127.0.0.1:<port> --protocol <name>
```

Additional options are defined in `serve` when the stack is built. The
on-disk representation defines precisely how the model will be served.

`serve`:

- serves the model at that address with the specified protocol. It
  stays in the foreground, answering requests that name
  `<model-name>`, until it receives `SIGTERM`.
- writes exactly one line to standard output: `ready` once the model
  is loaded and accepting requests, or `error <code> <message>` if it
  cannot serve the model, after which it exits non-zero.
- writes everything else, including the server logs, to standard
  error.

The `<code>` is one of:

| Code | Meaning |
|---|---|
| `protocol` | the server does not speak the protocol |
| `address` | the address cannot be listened on |
| `weights` | the model files are missing or will not load |
| `memory` | the host does not have enough memory |
| `host` | the host doesn't meet some other requirement: an accelerator, a driver, an operating system version |
| `other` | catch-all for other errors |

In practice `serve` is a short script that the builder generates. For
the model above, the core of the script might look like this:

```sh
"$here/llama-server" --model "$here/../qwen3-8b-q4_k_m.gguf" \
  --alias qwen3-8b --ctx-size 32768 \
  --host "$host" --port "$port" >&2 &
```

In addition, the script needs to:

1. Parse its two arguments (`--listen` and `--protocol`).
2. Refuse a protocol that llama-server doesn't support.
3. Wait for the server's `/health` endpoint before printing `ready`.
4. Pass `SIGTERM` on to the server when the stack is torn down.

#### `check`

Models can be large and take a long time to start. To save time in
failure scenarios, a self-contained model's directory MAY also contain a
`bin/check` executable, which can perform some fast checks to determine
if the model's requirements are met. It takes a single argument
(`--protocol <name>`) and writes one line to standard output:

- `ok` when the host can serve the model as built.
- `error <code> <message>` (similar to `serve`) when the model's
  requirements are not met.

When `bin/check` is present, a launcher SHOULD run it before `serve`.
A consumer or tool can also run it without launching anything.

#### `endpoint.json`

An externally-hosted model's directory contains an `endpoint.json` file
in the following format:

```json
{
  "url": "https://models.example.com/v1",
  "model": "qwen3-coder-30b-a3b",
  "protocols": ["anthropic-messages", "openai-chat-completions"]
}
```

- `url` — the base URL the model is served at.
- `model` — the name of the model to request.
- `protocols` — the protocols the endpoint speaks (see
  [§1](#1-model-gateways-and-protocols)).

`endpoint.json` SHOULD NOT hold a credential. Secrets are not yet part
of the spec; this remains an open question (3).

NOTE: `endpoint.json` is expected to be folded into the `model.json`
every model directory will hold, as its `endpoint` object with these
three fields unchanged. Until that file is specified, this proposal
keeps `endpoint.json` (open question 6).

### 3. Build

A self-contained model's files are a source (§10.1) that should be
pinned by digest when a stack is built. To minimize the cost of adding
a large self-contained model to a stack:

- A stack SHOULD NOT download a model at runtime.
- A builder SHOULD store each model file once per digest, so that
  every stack with the same self-contained model links to the same
  file.
- A launcher SHOULD NOT copy model files at launch (e.g. from the
  dependency set into a cache).

A model's declaration specifies the inference server that should be
used. For example, a GGUF file runs on llama.cpp or Ollama, and an MLX
model on oMLX or mlx-lm.

1. A builder SHOULD resolve the named server to a package and link it
   into the model directory, so that the stack is self-contained.
2. A builder that cannot resolve the named server SHOULD fail the
   build, naming the model and the server.
3. A builder that knows the server cannot load the model files' format
   SHOULD fail the build, naming the model and the server.
4. A builder SHOULD generate `serve`, and `check` where it can. These
   executables SHOULD include every serving parameter in them: the
   model files' path, the context length, offload, concurrency.
5. For an externally-hosted model, a builder SHOULD write
   `endpoint.json` from the declaration.

If a serving parameter changes, the stack must be built again.

### 4. Declaration

§6.1 gains two optional fields:

- `models` — the models the agent client can use.
- `defaultModel` — the name of the model the agent client uses by
  default. If only one model is defined, the builder SHOULD default to
  that one. Otherwise, the builder SHOULD fail if there are multiple
  models and no default is specified.

Each model has a `name`, which is its directory under
`share/agent-models/`. A declaration SHOULD be explicit about whether a
model is self-contained or externally-hosted:

- **Self-contained.** The model is part of the stack's dependency set,
  and the stack serves the model on launch, with no network access
  required. A self-contained model is declared by:
  - `source` — where the model files come from, pinned by revision and
    content digest. Source does not have a defined schema.
  - `server` — the name of the package that serves the model files.
- **Externally-hosted.** The model is not part of the stack's
  dependency set; it is hosted at the defined endpoint. An
  externally-hosted model is declared by `url`, `model`, and
  `protocols`, which the builder writes to `endpoint.json`.

### 5. Deployment patterns

A model can be served in one of the following ways:

| Where the model is served | Managed by | Type |
|---|---|---|
| Local host | Agent stack ([Launch](#6-launch)) | Self-contained |
| On the host | You | Externally-hosted, with a loopback `url` |
| On another machine | Someone else | Externally-hosted |
| On a third-party platform | Vendor | Externally-hosted |

One stack can contain multiple models with different deployment
patterns. For example, a stack can contain a self-contained model and
reference an externally-hosted model from a team server.

### 6. Launch

The stack owns the lifecycle of every server it starts. A launcher that
serves models SHOULD have the following properties:

1. **No manual setup.** Externally-hosted models require an existing
   inference server, but otherwise there should be no dependencies on
   the underlying host system beyond hardware and drivers.
2. **No manual cleanup.** When a stack is torn down or killed, all of
   its processes (including inference servers) are torn down too.
3. **Multiple launches of a stack can share servers.** The first launch
   starts the servers, and concurrent launches of the same stack share
   those servers. Sharing saves memory and keeps caches warm. Once all
   launches of a stack exit, the servers are torn down.
4. **Each model has its own server.** If a stack has multiple
   self-contained models, the stack SHOULD start a separate server for
   each one at launch.
5. **Each server should use a dynamically-assigned port.** To avoid
   port conflicts, servers should not use fixed ports.
6. **Server state is per-stack.** Some servers default to writing
   models, cache, and configuration to a user's home directory; this
   data SHOULD instead be written to a per-stack state directory. This
   state SHOULD NOT persist between launches.

When a stack has a model, the following launch process should be
followed (this alters the process defined in §7, between the steps of
resolving the agent and executing it):

1. **Start or join the servers.** For each self-contained model with no
   server running for this stack, run `check` if it exists, choose a
   loopback address and a free port, run `serve` with those arguments
   and a protocol the agent client speaks, and wait for the server to
   be ready. If the stack is already running via another launch, join
   the servers that are already running. For an endpoint found as
   described in [Finding the endpoint](#finding-the-endpoint), confirm
   that it is accepting requests, lists the model, and speaks a
   protocol the agent client speaks (§1). If any of these steps fails,
   the stack SHOULD refuse to launch and show the error to the user.
2. **Give the models to the agent client**, using the agent client's
   configuration (an environment variable, a generated file in the
   per-launch staging directory) to set the model. When there are
   multiple models in a stack, `defaultModel` is the one to start with.
3. **Manage server lifetimes** as defined above, even if the agent
   client crashes. This requires a wrapper or service manager, which
   amends §7. For example:

   ```sh
   <serve wrapper> -- <agent client> [args]
   ```

   The wrapper starts or joins the servers, runs the agent client, and
   stops the servers once the last launch exits.

#### Finding the endpoint

A model the stack does not serve itself is reached at an endpoint.
Launch looks for one in this order, and the first source that names an
endpoint wins; a later source is not merged into it:

1. **Flags.** `--model-url <url>` names the endpoint,
   `--model-api <protocol>` the protocol to use, and `--model <id>`
   the model to start with. For one-off runs and scripts.
2. **Environment.** An environment that runs a model gateway as a
   service sets these beside it, which makes this the main path:
   - `AGENT_STACKS_MODEL_URL` — the endpoint.
   - `AGENT_STACKS_MODEL_PROTOCOLS` — the protocols it speaks, as a
     comma-separated list, like `protocols` in `endpoint.json`.
   - `AGENT_STACKS_MODEL` — the model to start with.
   - `AGENT_STACKS_MODELS_DIR` — the directory the gateway serves
     model files from, so launch can list models without asking it.
3. **The stack.** The `endpoint.json` of each externally-hosted model
   under `share/agent-models/`.
4. **Asking.** With none of the above, launch sends `GET /v1/models`
   to loopback at the default ports of known model gateways (9931,
   8000, 11434, and LM Studio's 1234) and uses the first that answers.

Every source can name several models, which a single model variable
could not. Every model the endpoint lists is offered to the agent
client; `--model` and `AGENT_STACKS_MODEL` only choose the one to
start with. With several models and none chosen, the agent client's
own model picker decides.

`--model-api` is singular on purpose. It means "use this one", where
the lists in the environment and in `endpoint.json` say what the
endpoint can do and leave the choice to the matching in §1.

When a source names an endpoint but not its protocols, and always when
launch found it by asking, launch does not know what the endpoint
speaks: no model gateway reports it, and `GET /v1/models` lists models,
not APIs. See open question 5.

### 7. Bindings

As a Nix function argument:

```nix
mkAgentStack {
  name = "caveman-stack";
  harness = claude-code;
  plugins = [ agent-plugin-caveman ];
  models = [ agent-model-qwen3-8b ];
}
```

The model's source and server live in the model's package, and the
stack expression only names it. With one model, that model is also the
default.

As an environment manifest:

```toml
[install]
claude-code.pkg-path          = "claude-code"
agent-plugin-caveman.pkg-path = "agent-stacks/agent-plugin-caveman"
agent-model-qwen3-8b.pkg-path = "agent-stacks/agent-model-qwen3-8b"
```

An externally-hosted model is installed the same way, from a package
holding only its `endpoint.json`. That is how an organization can
publish the endpoints its stacks use.

## Open questions

1. **Context length.** The context length is fixed in `serve` when the
   stack is built. How does the agent client find out what the context
   length is so it knows how to avoid context overflows? Every model
   gateway reports a model's context window somewhere else (Appendix
   A), so it is not part of the contract in §1.
2. **Lazy loading.** To make it faster to launch a stack, should we
   consider only starting a server when an agent client makes a
   request to it?
3. **Secrets.** In the current proposal, `endpoint.json` doesn't
   include secrets, so if the endpoint for an externally-hosted model
   requires secrets, we have no way of letting the agent client
   authenticate. How do we let agent clients authenticate without
   leaking secrets?
4. **Single base URL.** Some agent clients, such as Claude Code, take a
   single base URL. How do they reach models at different URLs?
5. **Which protocols does an endpoint speak?** No model gateway reports
   it. When launch finds an endpoint by asking, or a source gives no
   protocols, launch has to recognise the gateway by its own endpoints
   (llama.cpp's `/props`, Ollama's `/api/ps`, oMLX's
   `/v1/models/status`) or make a test call per protocol. Is that
   fallback part of this proposal, or left to each launcher?
6. **`model.json`.** A model directory will hold a `model.json` that
   describes the model: its format, its files and their digests, its
   source, its context length. Its schema waits on a spike that feeds
   each inference server from it with links alone. `endpoint.json` is
   expected to become its `endpoint` object, fields unchanged, so that
   a model has either model files or an endpoint and never both. Does
   that fold work for externally-hosted models as written here, and
   what else does a remote model's `model.json` need to say?

## Appendix A: Model gateways that meet §1

This appendix is non-normative, and grows with each agent client and
model gateway tested. Each server below was read from its source at
the commit named, in November 2026; none was run for this table.

| | llama.cpp `llama-server` | oMLX | Ollama |
| --- | --- | --- | --- |
| Source read | ggml-org/llama.cpp `6d54465e` | jundot/omlx `633d6cf6` | ollama/ollama `8ae8acb8` |
| `anthropic-messages` | yes | yes | yes |
| `openai-chat-completions` | yes | yes | yes |
| `open-responses` | yes | yes | yes |
| Every model at one endpoint | in router mode, started without `-m` | yes | yes |
| `GET /v1/models` lists models not loaded | yes | yes | yes |
| Loads the model a request names | yes, unless `--no-models-autoload` | yes, evicting the least recently used | yes |
| Default port | 9931 | 8000 | 11434 |
| Reports context window at | `/props` | `/v1/models/status` | `/api/ps` |

The last row is why context length is still an open question: each
server reports it somewhere else.

On the agent client side, Claude Code speaks `anthropic-messages`.
Codex speaks only `open-responses`. opencode and pi speak
`openai-chat-completions`, and pi also `anthropic-messages`. A
gateway that speaks only `openai-chat-completions`, such as
`mlx_lm.server`, needs a router in front for Claude Code or Codex.
