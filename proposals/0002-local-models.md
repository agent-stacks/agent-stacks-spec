# Proposal 0002: Local models

- Status: Draft
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.3 and related sections
  (§5.1, §6.1, §6.4, §7, §8, §9, §10). Contains references to
  [Proposal 0001](0001-host-sandbox.md)
  ([#4](https://github.com/agent-stacks/agent-stacks-spec/pull/4)).
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
   hosted by the stack has its weights in that directory. A
   model hosted elsewhere is represented as an `endpoint.json` file
   saying where the model is hosted.
2. **Build.** For models hosted by the stack, the model file is fetched
   when the stack is built, and digest-pinned for reproducibility. The
   builder resolves the server, and links it similar to how §8 resolves
   a plugin's runtimes. The build SHOULD fail if either of these steps
   fails.
3. **Declaration.** The vocabulary in §6.1 defines `models`, either
   self-hosted (carried, per §6.2) or externally-hosted (borrowed), and
   `defaultModel`, the model used by the agent client.
4. **Launch.** The stack owns the lifecycle of every inference server
   it starts. When you launch a stack, each inference server starts on
   a loopback port, which gives the agent client access to the models.
   These servers stop after the last agent client has exited. The
   launch will fail if a declared model can't be served.

## Motivation

The choice of model has perhaps the biggest impact on how an agent
stack behaves. By making models part of the definition of a stack, we
can make stacks more self-contained and reproducible.

### Use cases

This proposal is informed by two real-world use cases:

**Data privacy.** A person or a team wants an agent to have access to
data without sending it to a third party. This could be medical or
financial records, a private codebase, or company IP. These users will
accept a less capable model in exchange for keeping data on their own
machines.

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
   (in the self-hosted case) or accessed (in the externally-hosted
   case), the stack will fail to launch.
4. **Models are not responsible for enforcing the behavior of the agent
   client.** If a stack declares a local model, and the agent client
   uses a cloud model instead, the only way to prevent this behavior is
   by defining a properly-configured sandbox as part of the stack.
   Notably, the use of a sandbox is the only way to guarantee that data
   is not being leaked to a third party.

### Externally-hosted models

Open models are increasingly run somewhere other than the user's own
machine:

1. A virtual machine running in the cloud.
2. A GPU server in an organization's on-prem datacenter.
3. A third-party provider that hosts open models (e.g. OpenRouter).

An agent stack should be able to work with these externally-hosted
models too. These models are represented in the stack as an
`endpoint.json` file saying where the model is hosted. See
[Deployment patterns](#4-deployment-patterns) for more.

### Self-hosted models

A model self-hosted by a stack consists of:

1. File(s) containing the model's weights (e.g. GGUF, MLX).
2. The inference server (e.g. llama.cpp, Ollama, oMLX, vLLM) and its
   dependencies (e.g. CUDA).
3. A `bin/serve` executable that starts the server.

NOTE: A self-hosted model MAY have specific hardware requirements that
limit what machines it can run on.

## Out of scope

- **Model routing.** A stack can give its agent client several models.
  However, a stack only ever defines a single default; automatically
  routing a request to the best model is not supported.
- **Egress enforcement.** This proposal does nothing to limit an agent
  client's network access. To enforce a particular egress policy, you
  must use a sandbox (Proposal 0001).
- **Management of externally-hosted models.** Externally-hosted models
  may be accessed by the agent client in an agent stack, but an agent
  stack does not otherwise manage those models or their inference
  servers.
- **Model distribution.** The only requirement is that self-hosted
  models have pinnable weights. Model registries and download processes
  are out of scope.
- **Context management.** Local models usually run with much smaller
  context windows than hosted models do. As a result, good
  context window management is critical, but how this is done is out
  of scope for this proposal.

## Proposal

### 1. Layout

1. A stack MAY store models at
   `<stack root>/share/agent-models/<model-name>/`, one directory per
   model. Model names SHOULD be unique within a stack.
2. A self-hosted model's directory SHOULD contain the model's weights
   and a `bin/` folder containing an executable named `serve` and a
   link to the server it runs. The `bin/` folder MAY also contain an
   executable named `check`.
3. An externally-hosted model's directory SHOULD contain one file,
   `endpoint.json`.
4. The weights and the server SHOULD be links into the stack's
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
        ├── qwen3-8b/                  self-hosted
        │   ├── bin/
        │   │   ├── serve              the serve contract
        │   │   ├── check              optional host check
        │   │   └── llama-server       link to the server it runs
        │   └── qwen3-8b-q4_k_m.gguf   link to the weights
        └── team-coder/                externally-hosted
            └── endpoint.json
```

The `agent-models/` directory shows you every model in the stack. If a
stack does not have an `agent-models/` directory, it runs exactly the
same as it does today.

#### `serve`

`serve` is the interface between a launcher and a self-hosted model.
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
| `weights` | the weights are missing or will not load |
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
failure scenarios, a self-hosted model's directory MAY also contain a
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
  "url": "https://models.example.com",
  "model": "qwen3-coder-30b-a3b",
  "protocols": ["anthropic-messages", "openai-chat-completions"]
}
```

- `url` — the base URL the model is served at.
- `model` — the name of the model to request.
- `protocols` — the protocols the endpoint supports (see Appendix A).

`endpoint.json` SHOULD NOT hold a credential. Secrets are not yet part
of the spec; this remains an open question (4).

### 2. Build

A self-hosted model's weights are a source (§10.1) that should be
pinned by digest when a stack is built. To minimize the cost of adding
a large self-hosted model to a stack:

- A stack SHOULD NOT download a model at runtime.
- A builder SHOULD store weights once per digest, so that every stack
  with the same self-hosted model links to the same file.
- A launcher SHOULD NOT copy weights at launch (e.g. from the
  dependency set into a cache).

A model's declaration specifies the inference server that should be
used. For example, a GGUF file runs on llama.cpp or Ollama, and an MLX
model on oMLX or mlx-lm.

1. A builder SHOULD resolve the named server to a package and link it
   into the model directory, so that the stack is self-contained.
2. A builder that cannot resolve the named server SHOULD fail the
   build, naming the model and the server.
3. A builder that knows the server cannot load the weights' format
   SHOULD fail the build, naming the model and the server.
4. A builder SHOULD generate `serve`, and `check` where it can. These
   executables SHOULD include every serving parameter in them: the
   weights' path, the context length, offload, concurrency.
5. For an externally-hosted model, a builder SHOULD write
   `endpoint.json` from the declaration.

If a serving parameter changes, the stack must be built again.

### 3. Declaration

§6.1 gains two optional fields:

- `models` — the models the agent client can use.
- `defaultModel` — the name of the model the agent client uses by
  default. If only one model is defined, the builder SHOULD default to
  that one. Otherwise, the builder SHOULD fail if there are multiple
  models and no default is specified.

Each model has a `name`, which is its directory under
`share/agent-models/`. A declaration SHOULD be explicit about whether a
model is self-hosted or externally-hosted:

- **Self-hosted.** The model is part of the stack's dependency set, and
  the stack serves the model on launch, with no network access
  required. A self-hosted model is declared by:
  - `source` — where the weights come from, pinned by revision and
    content digest. Source does not have a defined schema.
  - `server` — the name of the package that serves the weights.
- **Externally-hosted.** The model is not part of the stack's
  dependency set; it is hosted at the defined endpoint. An
  externally-hosted model is declared by `url`, `model`, and
  `protocols`, which the builder writes to `endpoint.json`.

### 4. Deployment patterns

A model can be served in one of the following ways:

| Where the model is served | Managed by | Type |
|---|---|---|
| Local host | Agent stack ([Launch](#5-launch)) | Self-hosted |
| On the host | You | Externally-hosted, with a loopback `url` |
| On another machine | Someone else | Externally-hosted |
| On a third-party platform | Vendor | Externally-hosted |

One stack can contain multiple models with different deployment
patterns. For example, a stack can contain a self-hosted model and
reference an externally-hosted model from a team server.

### 5. Launch

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
   self-hosted models, the stack SHOULD start a separate server for
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

1. **Start or join the servers.** For each self-hosted model with no
   server running for this stack, run `check` if it exists, choose a
   loopback address and a free port, run `serve` with those arguments
   and a protocol the agent client speaks, and wait for the server to
   be ready. If the stack is already running via another launch, join
   the servers that are already running. For each externally-hosted
   model, confirm that the endpoint is accepting requests and uses a
   compatible protocol. If any of these steps fails, the stack SHOULD
   refuse to launch and show the error to the user.
2. **Give the models to the agent client**, using the agent client's
   configuration (an environment variable, a generated file in the
   per-launch staging directory) to set the model. When there are
   multiple models in a stack, `defaultModel` is the one to start with.
3. **Manage server lifetimes** as defined above, even if the agent
   client crashes. This requires a wrapper or service manager, which
   amends §7. For example, this is how it might look with a sandbox
   wrapper too:

   ```sh
   <serve wrapper> -- [<sandbox> [options] --] <agent client> [args]
   ```

### 6. Bindings

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
   length is so it knows how to avoid context overflows?
2. **Lazy loading.** To make it faster to launch a stack, should we
   consider only starting a server when an agent client makes a
   request to it?
3. **Interaction with sandboxes.** Some sandboxes may prevent a model
   from getting the hardware access it needs to run. And a sandbox may
   prevent an agent client from making requests to the model at all.
   How should we handle this?
4. **Secrets.** In the current proposal, `endpoint.json` doesn't
   include secrets, so if the endpoint for an externally-hosted model
   requires secrets, we have no way of letting the agent client
   authenticate. How do we let agent clients authenticate without
   leaking secrets?

## Appendix A: Protocol names

This appendix is non-normative, and grows with each agent client and
server tested. These protocol names are used by `serve --protocol`,
`check --protocol`, and the `protocols` field of `endpoint.json`. The
protocol is appended to an endpoint's URL.

| Name | API |
|---|---|
| `anthropic-messages` | Anthropic Messages API, `POST /v1/messages` |
| `openai-chat-completions` | OpenAI Chat Completions API, `POST /v1/chat/completions` |
| `open-responses` | [Open Responses](https://openresponses.org/), the open specification of OpenAI's Responses API, `POST /v1/responses` |

Claude Code speaks `anthropic-messages`. llama-server serves
`openai-chat-completions`, and `anthropic-messages`
[since January 2026](https://huggingface.co/blog/ggml-org/anthropic-messages-api-in-llamacpp).
