# Proposal 0002: Local models

- Status: Draft
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.3 and related sections
  (§5.1, §6.1, §6.2, §6.4, §7, §8, §9, §10). Its sandbox rule builds
  on [Proposal 0001](0001-host-sandbox.md).
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
   `<stack root>/share/agent-models/`, one directory per model, each
   with a `model.json` that describes it. A model hosted by the stack
   has its model files beside it, in `files/`. A model hosted
   elsewhere has no files, and its `model.json` says where the model
   is hosted.
2. **Build.** For models hosted by the stack, the model file is fetched
   when the stack is built, and digest-pinned for reproducibility. The
   builder resolves the gateway package that serves them, and links it
   similar to how §8 resolves a plugin's runtimes. The build SHOULD
   fail if either of these steps fails.
3. **Declaration.** The vocabulary in §6.1 defines `models`, either
   carried, borrowed, or externally-hosted, and `modelRoles`, the
   model the agent client uses for each of its roles.
4. **Launch.** A model gateway serves the models, and the launcher
   never starts one. A stack that carries models ships
   `bin/<stack>-gateway` to serve them, which the user or a service
   manager runs beside the launcher. When you launch a stack, the
   launcher finds the endpoint and gives the agent client access to
   the models; when the gateway is not up, it says so and names the
   command. The launch will fail if a declared model can't be served.

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
2. **The stack carries what serves its models.** If a model is hosted
   by the stack itself, there should be no external dependencies
   outside of the stack at runtime, other than hardware and drivers.
   The stack carries its model gateway and the command that runs it;
   the user or a service manager decides when it runs.
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
models too. These models are represented in the stack by a
`model.json` whose `endpoint` says where the model is hosted. See
[Deployment patterns](#5-deployment-patterns) for more.

### Self-contained models

A self-contained model consists of:

1. The model files (e.g. GGUF, MLX).
2. The inference server (e.g. llama.cpp, Ollama, oMLX, vLLM) and its
   dependencies (e.g. CUDA), wrapped in a gateway package
   ([§7](#7-gateway-packages)).
3. A `bin/<stack>-gateway` executable that serves them.

NOTE: A self-contained model MAY have specific hardware requirements
that limit what machines it can run on.

## Out of scope

- **Model routing.** A stack can give its agent client several models.
  However, a stack only ever defines a single default; automatically
  routing a request to the best model is not supported. A router that
  does so is a model gateway like any other
  ([§1](#1-model-gateways-and-protocols)), and its rules are its own
  configuration.
- **Egress enforcement.** This proposal does nothing to limit an agent
  client's network access, beyond letting a sandbox reach the model
  endpoint ([§6](#with-a-sandbox)). Proposal 0001 is where that limit
  is set.
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
a gateway package's `describe` ([§7](#7-gateway-packages)),
`AGENT_STACKS_MODEL_PROTOCOLS`, and the `protocols` field of a
`model.json`'s `endpoint` ([§2](#2-layout)). The list grows with each
agent client and model gateway tested; a name that is not in it
SHOULD NOT be used until it is added.

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
2. Every model's directory SHOULD contain a `model.json`
   ([below](#modeljson)) that describes the model.
3. A self-contained model's directory SHOULD also contain `files/`,
   holding the model files exactly as the model's `layout` arranges
   them. What serves them is the stack's gateway package, not the
   model ([§7](#7-gateway-packages)).
4. An externally-hosted model's directory SHOULD contain only its
   `model.json`, with an `endpoint` and no `files/`.
5. The model files SHOULD be links into the stack's dependency set,
   similar to how plugin runtimes work (§8.3).
6. `<model-name>` is how the stack refers to the model.
7. A stack that carries a model SHOULD also provide
   `<stack root>/bin/<stack-name>-gateway`, the stack's own model
   gateway ([§6](#6-launch)).

```text
caveman-stack/
├── bin/
│   ├── caveman-stack
│   └── caveman-stack-gateway          agent-gateway-llama-cpp serve
└── share/
    ├── agent-plugins/
    │   └── caveman/...
    └── agent-models/
        ├── qwen3-8b/                  self-contained
        │   ├── model.json
        │   └── files/
        │       ├── Qwen3-8B-Q4_K_M.gguf   link to the model file
        │       └── LICENSE
        └── team-coder/                externally-hosted
            └── model.json             with an endpoint, no files/
```

The `agent-models/` directory shows you every model in the stack. If a
stack does not have an `agent-models/` directory, it runs exactly the
same as it does today.

A borrowed model ([§4](#4-declaration)) has no directory here and no
`model.json`. It stays in the consumer's Hugging Face cache, and
launch and the model gateway read that cache directly, as they do
today.

#### `model.json`

`model.json` says what a model is and, for a model with files, how
they are stored. A gateway package reads it to feed its inference
server with links alone ([§7](#7-gateway-packages)), and launch reads
it to match the model to the agent client. The same schema describes
a local and an externally-hosted model.

Where CNCF [ModelPack](https://github.com/modelpack/model-spec) has a
name for a fact, `model.json` uses it, under ModelPack's `descriptor`
and `config`. Where ModelPack has no field, or its shape is too loose
to check against, the field is ours.

| Field | Meaning | From |
| --- | --- | --- |
| `schemaVersion` | `1` | ours |
| `id` | the model's ID, which every model gateway serves it as | ours |
| `layout` | how `files/` is arranged | ours |
| `format` | what the model files are | ModelPack's name, at the top level |
| `entry` | what an inference server loads, relative to the model's directory | ours |
| `source` | where the files came from: `hub`, `repo`, `revision` | ours |
| `descriptor` | `family`, `licenses`, and ModelPack's other descriptor fields | ModelPack |
| `config.architecture`, `config.paramSize` | the architecture and the counted parameter size | ModelPack |
| `config.quantization` | the quantization, as an object | ModelPack's name, our shape |
| `config.contextLength` | the context window the model was trained for | ours |
| `config.capabilities` | what the model can do, such as `tool-use` and `vision` | ours (open question 6) |
| `files[]` | each model file: `path`, `role`, `sha256`, `size` | ours, OpenSSF Model Signing digests |
| `endpoint` | where an externally-hosted model is served: `url`, `model`, `protocols` | ours |

Field names are camelCase throughout.

##### Files or endpoint, never both

A `model.json` has either the file fields (`layout`, `format`,
`entry`, `files`) or `endpoint`, never both. `schemaVersion`, `id`,
`descriptor` and `config` are common to both, so launch knows the
context window and capabilities of a remote model as it does of a
local one.

A local model never names an endpoint. Its URL is not known when the
stack is built: which model gateway serves it, and on which port, is
decided when it runs, by the gateway package and by
[Finding the endpoint](#finding-the-endpoint).

##### `id`

`id` is the model's API ID: the `id` that `GET /v1/models` lists and
a request's `model` names, on every model gateway that serves it
([§7](#7-gateway-packages)). So:

1. It SHOULD be unique among the models an environment serves, and
   SHOULD equal `<model-name>`, the model's directory name. A gateway
   package's `check` and `serve` name a model by its `id`
   ([§7](#7-gateway-packages)), so when the two differ, the name in
   their output is not the directory a user looks for.
2. It MUST NOT contain `/` or `:`. Gateways turn it into a file name
   or a section name, and some give both characters a meaning of
   their own.
3. A `model.json` with no `id` is named by its directory.

##### `layout`, `format` and `entry`

`layout` says how `files/` is arranged. It is an open list; a new
store is a new value, not a change to this schema:

- `plain` — the repository snapshot as published.
- `hf-cache` — a Hugging Face cache tree,
  `models--<org>--<repo>/{refs,blobs,snapshots}`, with `refs/main`.
- `ollama` — Ollama's manifests and blobs.

`format` says what the files are: `gguf`, `mlx`, `safetensors`,
`onnx`, and so on. It is independent of the layout, and it decides
which inference servers can load the model. A gateway bridges any
layout with links; nothing bridges a format its server cannot read.

`entry` is what an inference server loads, relative to the model's
directory, so it includes `files/` and any layout prefix: the
directory for MLX, and for GGUF the file, or the first shard of a
split file.

##### `files[]`

Every model file has an entry:

- `path` — relative to the directory of `entry` (the directory
  itself when `entry` is one), whatever the layout. For `hf-cache`
  that is the snapshot directory.
- `role` — one of the roles below.
- `sha256` — the file's digest, which pins it.
- `size` — its size in bytes.

The `path` and `sha256` pairs are the digest list an OpenSSF
[Model Signing](https://github.com/sigstore/model-transparency)
manifest holds, so a model directory is one step from a signed one.
Signing itself is out of scope.

| Role | The file |
| --- | --- |
| `weights` | tensors: a GGUF file or shard, a safetensors file |
| `config` | the model's configuration, such as `config.json` |
| `tokenizer` | the tokenizer and its configuration |
| `index` | the index of a sharded checkpoint |
| `licence` | the licence |
| `chat-template` | a chat template outside the weights or tokenizer |
| `mmproj` | a vision projector, which llama.cpp loads beside the GGUF |
| `draft` | a draft model for speculative decoding |
| `other` | anything else |

A gateway finds the files its server needs by role. llama.cpp, for
one, needs `mmproj` beside the GGUF it loads.

##### `config.quantization` and `config.capabilities`

`quantization` is an object, so that it can be checked:

- GGUF: `{ "type": "Q4_K_M" }`, from the header's `general.file_type`.
- MLX and safetensors: `{ "bits": 4, "groupSize": 64 }`, from
  `config.json`.

`capabilities` is a list of names: `tool-use` when the chat template
supports tool calls, `vision` when the model has a vision projector or
a vision configuration. Launch uses it, with `contextLength`, to check
a model against what the agent client needs, and to hide one that
cannot serve it.

##### Facts come from the files

A builder SHOULD write `model.json` from the model files when it
builds the model, never by hand and never from the Hub API:

- GGUF: from the header (`general.architecture`, `general.file_type`,
  `<arch>.context_length`, `general.size_label`).
- MLX and safetensors: from `config.json`.

The Hub API can disagree with the files. For Qwen2.5-0.5B-Instruct's
GGUF it gives a context length of 8192, where the header says 32768,
and 32768 is what llama.cpp serves. `paramSize` is the counted size,
which can differ from the name a model is sold under.

##### Examples

A GGUF model, `plain` layout:

```json
{
  "schemaVersion": 1,
  "id": "qwen3-8b",
  "layout": "plain",
  "format": "gguf",
  "entry": "files/Qwen3-8B-Q4_K_M.gguf",
  "source": {
    "hub": "huggingface",
    "repo": "Qwen/Qwen3-8B-GGUF",
    "revision": "<commit>"
  },
  "descriptor": {
    "family": "qwen3",
    "licenses": ["Apache-2.0"]
  },
  "config": {
    "architecture": "qwen3",
    "paramSize": "8b",
    "quantization": { "type": "Q4_K_M" },
    "contextLength": 32768,
    "capabilities": ["tool-use"]
  },
  "files": [
    { "path": "Qwen3-8B-Q4_K_M.gguf", "role": "weights",
      "sha256": "<sha256>", "size": 5027783488 },
    { "path": "LICENSE", "role": "licence",
      "sha256": "<sha256>", "size": 11343 }
  ]
}
```

An MLX model, `plain` layout:

```json
{
  "schemaVersion": 1,
  "id": "qwen3-coder-30b-a3b-4bit",
  "layout": "plain",
  "format": "mlx",
  "entry": "files/",
  "source": {
    "hub": "huggingface",
    "repo": "mlx-community/Qwen3-Coder-30B-A3B-Instruct-4bit",
    "revision": "<commit>"
  },
  "descriptor": {
    "family": "qwen3",
    "licenses": ["Apache-2.0"]
  },
  "config": {
    "architecture": "qwen3_moe",
    "paramSize": "30b",
    "quantization": { "bits": 4, "groupSize": 64 },
    "contextLength": 262144,
    "capabilities": ["tool-use"]
  },
  "files": [
    { "path": "config.json", "role": "config",
      "sha256": "<sha256>", "size": 1102 },
    { "path": "tokenizer.json", "role": "tokenizer",
      "sha256": "<sha256>", "size": 11422654 },
    { "path": "tokenizer_config.json", "role": "tokenizer",
      "sha256": "<sha256>", "size": 9711 },
    { "path": "chat_template.jinja", "role": "chat-template",
      "sha256": "<sha256>", "size": 6116 },
    { "path": "model.safetensors.index.json", "role": "index",
      "sha256": "<sha256>", "size": 98412 },
    { "path": "model-00001-of-00004.safetensors", "role": "weights",
      "sha256": "<sha256>", "size": 5363922944 },
    { "path": "model-00002-of-00004.safetensors", "role": "weights",
      "sha256": "<sha256>", "size": 5366734592 },
    { "path": "model-00003-of-00004.safetensors", "role": "weights",
      "sha256": "<sha256>", "size": 5366734664 },
    { "path": "model-00004-of-00004.safetensors", "role": "weights",
      "sha256": "<sha256>", "size": 1086327552 },
    { "path": "LICENSE", "role": "licence",
      "sha256": "<sha256>", "size": 11343 }
  ]
}
```

An externally-hosted model, with no `files/`:

```json
{
  "schemaVersion": 1,
  "id": "team-coder",
  "config": {
    "contextLength": 131072,
    "capabilities": ["tool-use"]
  },
  "endpoint": {
    "url": "https://models.example.com/v1",
    "model": "qwen3-coder-30b-a3b",
    "protocols": ["anthropic-messages", "openai-chat-completions"]
  }
}
```

`endpoint` is the `endpoint.json` of an earlier draft, folded in with
its fields unchanged:

- `url` — the base URL the model is served at.
- `model` — the name of the model to request.
- `protocols` — the protocols the endpoint speaks (see
  [§1](#1-model-gateways-and-protocols)).

`id` is how the stack and the agent client name the model; `model` is
what the remote gateway calls it. `endpoint` SHOULD NOT hold a
credential. Secrets are not yet part of the spec; this remains an open
question (3).

### 3. Build

A self-contained model's files are a source (§10.1) that should be
pinned by digest when a stack is built. To minimize the cost of adding
a large self-contained model to a stack:

- A stack SHOULD NOT download a carried model at runtime. A borrowed
  model ([§4](#4-declaration)) is the exception: the consumer downloads
  it into their own cache, and a model gateway MAY fetch it there, so
  launch announces it instead.
- A builder SHOULD store each model file once per digest, so that
  every stack with the same self-contained model links to the same
  file.
- A builder SHOULD write each carried model's `model.json` when it
  builds the model, reading its facts from the files, and record each
  file's digest in its `files[]` ([§2](#modeljson)).
- A launcher SHOULD NOT copy model files at launch (e.g. from the
  dependency set into a cache).

A stack's declaration names the gateway package that serves its
carried models. For example, a GGUF file runs on llama.cpp or Ollama,
and an MLX model on oMLX or mlx-lm.

1. A builder SHOULD resolve the named gateway package and link it into
   the stack's dependency set, so that the stack is self-contained,
   and write `bin/<stack-name>-gateway` to run its `serve` over the
   stack's own `share/agent-models/`.
2. A builder that cannot resolve the named gateway package SHOULD fail
   the build, naming the stack and the gateway.
3. A builder SHOULD run the gateway's `check` on every carried model,
   and fail the build on a model the gateway skips with `format`,
   `platform`, `layout`, `weights` or `model-json`
   ([Codes](#codes)), naming the model, the gateway, and the reason.
   It reads the codes, not `check`'s exit status, which is 0 when
   models are skipped.
4. A builder SHOULD put every serving parameter the stack decides (the
   context length, offload, concurrency) in `bin/<stack-name>-gateway`,
   passed through to the server.
5. For an externally-hosted model, a builder SHOULD write a
   `model.json` with an `endpoint` from the declaration.

If a serving parameter changes, the stack must be built again.

### 4. Declaration

§6.1 gains three optional fields:

- `models` — the models the agent client can use.
- `gateway` — the gateway package ([§7](#7-gateway-packages)) that
  serves the stack's carried models. A builder SHOULD fail when a
  stack carries a model and names no gateway.
- `modelRoles` — the model the agent client uses for each of its
  roles (see [Model roles](#model-roles)). Every agent client has the
  role `default`, the model it starts with. If only one model is
  defined, the builder SHOULD make it the default. Otherwise, the
  builder SHOULD fail if there are multiple models and no default is
  specified.

Each model has a `name`, which is its directory under
`share/agent-models/`. A declaration SHOULD be explicit about whether a
model is self-contained or externally-hosted:

- **Self-contained.** The model is part of the stack's dependency set,
  and the stack serves the model on launch, with no network access
  required. A self-contained model is declared by:
  - `source` — where the model files come from, pinned by revision and
    content digest. The builder writes the revision to `model.json`'s
    `source`, and each file's digest to its `files[]`.

  The server is not part of a model's declaration. The stack's
  `gateway` serves every carried model.
- **Externally-hosted.** The model is not part of the stack's
  dependency set; it is hosted at the defined endpoint. An
  externally-hosted model is declared by `url`, `model`, and
  `protocols`, which the builder writes to `model.json`'s `endpoint`.

#### A model is carried or it is borrowed

A model that runs locally comes from somewhere, as the agent client
does in §6.2, and a declaration SHOULD make that explicit:

- **Carried.** The model files are part of the stack's dependency set.
  A self-contained model is carried, and the stack runs the same model
  on every machine it reaches.
- **Borrowed.** The model files are resolved from the consumer's
  Hugging Face cache (`HF_HUB_CACHE`, by default
  `~/.cache/huggingface/hub`) when the stack runs: whatever has been
  downloaded there, for example with `hf download`, at whatever
  revision. A borrowed model has no directory under
  `share/agent-models/` and no `model.json`. The model gateway serves
  it from the cache, and launch finds it by the ID the gateway lists.
  It is declared by the repository it comes from, with no digest.

Both are legitimate. Borrowing is how a consumer tries a model before
anyone packages it. A launcher SHOULD announce every borrowed model it
offers, on every launch, naming where it found it, as §9.3 asks for an
agent client resolved from the consumer's environment. An
externally-hosted model is neither carried nor borrowed: it has no
model files.

#### Model roles

Some agent clients use more than one model in a session. Claude Code,
for one, picks a model by task: its `opus`, `sonnet`, and `haiku`
aliases, and a model for subagents. `modelRoles` maps each role to a
model, in the agent client's own words:

| Agent client | Roles | Set at launch through |
| --- | --- | --- |
| Claude Code | `default`, `opus`, `sonnet`, `haiku`, `subagent` | `--model`, `ANTHROPIC_DEFAULT_OPUS_MODEL`, `ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`, `CLAUDE_CODE_SUBAGENT_MODEL` |
| Any other | `default` | the agent client's own model setting |

1. A role names a model in `models`. A builder SHOULD fail when it
   names one that is not there, naming the role and the model.
2. An agent client with no roles takes only `default`. A launcher
   SHOULD warn about a role the stack's agent client does not have,
   and otherwise ignore it.
3. A launcher SHOULD set every role the declaration leaves out to the
   default model, so that no role falls back to a model the stack does
   not name. For a stack declared for privacy, that fallback is
   usually a cloud model.

This amends §6.4, which says a declaration does not name per-agent
configuration. A stack names one agent client (§5.4), so its
declaration MAY say which model that agent uses for each of its roles.
The role names are the agent client's; the flags and variables that
set them stay with the launch (§7), as §6.4 intends.

### 5. Deployment patterns

A model can be served in one of the following ways:

| Where the model is served | Gateway run by | Type |
| --- | --- | --- |
| On the host, by the stack's own gateway | You or a service manager, with `bin/<stack>-gateway` ([Launch](#6-launch)) | Carried |
| On the host, by a gateway you already run | You (an app, a service in your environment) | Borrowed, or externally-hosted with a loopback `url` |
| On another machine | Someone else | Externally-hosted |
| On a third-party platform | Vendor | Externally-hosted |

In every pattern the launcher connects to a model gateway that is
already running. It never starts one, so the gateway's lifetime is not
tied to any one launch of the agent client.

One stack can contain multiple models with different deployment
patterns. For example, a stack can contain a self-contained model and
reference an externally-hosted model from a team server.

### 6. Launch

A stack that carries models has two executables, one for each side:

- `bin/<stack-name>-gateway` serves the carried models at one endpoint
  and stays in the foreground until it receives `SIGTERM`. The user
  runs it, or a service manager does: a service in the consumer's
  environment, launchd, systemd.
- `bin/<stack-name>`, the launcher, runs the agent client against that
  endpoint.

The launcher never starts a model gateway, the stack's own or any
other. It connects to one that is running, or says that none is. A
launcher whose stack names a gateway package SHOULD record which one
(for example in `AGENT_STACKS_GATEWAY`), so that launch and other
tools can call its `describe` and `status`.

The stack's own gateway SHOULD have the following properties:

1. **No manual setup.** There should be no dependencies on the
   underlying host system beyond hardware and drivers.
2. **No manual cleanup.** When the gateway is stopped or killed, every
   process it started, including inference servers, stops too.
3. **Every launch shares it.** One gateway serves every launch of the
   stack, concurrent or not. Sharing saves memory and keeps caches
   warm, and a launch that exits leaves the gateway running.
4. **It listens where the launcher looks.** It takes `--listen <addr>`
   and otherwise listens on a loopback address the stack's launcher
   also knows. An inference server it starts behind it SHOULD use a
   dynamically-assigned port, to avoid port conflicts.
5. **Server state is per-stack.** Some servers default to writing
   models, cache, and configuration to a user's home directory; this
   data SHOULD instead be written to a per-stack state directory. This
   state SHOULD NOT persist between runs of the gateway.

When a stack has a model, the following launch process should be
followed (this alters the process defined in §7, between the steps of
resolving the agent and executing it):

1. **Find and check the endpoint**, as described in
   [Finding the endpoint](#finding-the-endpoint), and confirm that it
   is accepting requests, lists the models, and speaks a protocol the
   agent client speaks ([§1](#1-model-gateways-and-protocols)). When
   nothing answers, the launcher SHOULD refuse to launch, say that the
   model gateway is not up, and name the command that starts it, e.g.
   "the model gateway isn't up yet: run caveman-stack-gateway". If any
   other step fails, the stack SHOULD refuse to launch and show the
   error to the user.
2. **Give the models to the agent client**, using the agent client's
   configuration (an environment variable, a generated file in the
   per-launch staging directory) to set the model. When there are
   multiple models in a stack, the `default` role is the one to start
   with, and every other role is set as [Model roles](#model-roles)
   says.

Because the gateway outlives any one launch, the launcher needs no
wrapper or watcher: the launcher of spec §7 still hands over to the
agent client and gets out of the way.

#### With a sandbox

When the agent client runs inside a sandbox, as
[Proposal 0001](0001-host-sandbox.md) proposes:

1. The model gateway stays outside the sandbox. It is not the agent
   client or one of its children, and an inference server needs the
   accelerator, the memory, and the model files that a sandbox around
   the agent client has no reason to grant.
2. The launcher SHOULD add the endpoint's host and port to what the
   stack implies, the second input of Proposal 0001's effective
   configuration, and nothing more for the model: not the loopback
   interface as a whole, and not the gateway's other ports.
3. For an externally-hosted endpoint, that is its host name and port,
   which is the kind of rule both sandboxes in Proposal 0001 enforce.

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
     comma-separated list, like `protocols` in a `model.json`'s
     `endpoint`.
   - `AGENT_STACKS_MODEL` — the model to start with.
   - `AGENT_STACKS_MODELS_DIR` — the directories the gateway serves
     model files from, colon-separated, so launch can list models
     without asking it. A gateway package's `serve` reads the same
     variable ([§7](#serve)).
3. **The stack.** The `endpoint` in the `model.json` of each
   externally-hosted model under `share/agent-models/`. A launcher
   whose stack ships `bin/<stack-name>-gateway` sets
   `AGENT_STACKS_MODEL_URL` and `AGENT_STACKS_MODEL_PROTOCOLS` for its
   own gateway when they are not already set, so that gateway is found
   in step 2.
4. **Asking.** With none of the above, launch sends `GET /v1/models`
   to loopback at the default ports of known model gateways (9931,
   8000, 11434, and LM Studio's 1234) and uses the first that answers.

Every source can name several models, which a single model variable
could not. Every model the endpoint lists is offered to the agent
client; `--model` and `AGENT_STACKS_MODEL` only choose the one to
start with. With several models and none chosen, the agent client's
own model picker decides.

`--model-api` is singular on purpose. It means "use this one", where
the lists in the environment and in `model.json` say what the
endpoint can do and leave the choice to the matching in
[§1](#1-model-gateways-and-protocols).

When a source names an endpoint but not its protocols, launch asks the
gateway package's `describe` when it knows the package
([§7](#7-gateway-packages)). Otherwise, and always when launch found the
endpoint by asking, launch does not know what the endpoint speaks: no
model gateway reports it, and `GET /v1/models` lists models, not APIs.
See open question 5.

### 7. Gateway packages

A model gateway is wrapped for agent stacks by a package named
`agent-gateway-<name>`, one per inference server:
`agent-gateway-llama-cpp`, `agent-gateway-omlx`,
`agent-gateway-ollama`. The server itself is untouched; the package
gives a builder, a launcher, and other tools one small interface
instead of each server's own endpoints and flags. A new inference
server is a new package, not a new launcher.

This replaces the per-model `serve` and `check` of an earlier draft.
They move from each model to each gateway, which fits a gateway that
serves every model at one endpoint
([§1](#1-model-gateways-and-protocols)) and that the user runs
([§6](#6-launch)). It follows the `agent-sandbox-*` packages that carry
Proposal 0001's sandboxes: one package per backend, behind one contract.

1. A gateway package SHOULD provide one executable,
   `bin/agent-gateway-<name>`, with the four commands below.
2. It SHOULD pin the version of the server it wraps and carry the
   server in its dependency set.
3. It SHOULD run a contract test in its own build that exercises every
   command against the server it pins, as the `agent-sandbox-*`
   packages do.
4. It SHOULD stay thin. Anything the contract does not name is the
   server's own configuration, and `serve` passes it through.

Each command writes its result to standard output and everything else,
including the server's logs, to standard error.

#### `describe`

```sh
agent-gateway-<name> describe
```

Writes one JSON object, the gateway's profile, and starts nothing:

```json
{
  "formats": ["gguf"],
  "platforms": ["aarch64-darwin", "x86_64-linux", "aarch64-linux"],
  "version": "<pinned server version>",
  "protocols": ["anthropic-messages", "openai-chat-completions",
                "open-responses"],
  "defaultPort": 9931
}
```

- `formats` — the model file formats its inference server loads.
- `platforms` — the platforms the server runs on.
- `version` — the version of the server the package pins.
- `protocols` — the protocols
  ([§1](#1-model-gateways-and-protocols)) the server speaks. This is
  how launch learns what a gateway it knows speaks.
- `defaultPort` — where `serve` listens when not told otherwise. It is
  the server's own default, so it differs between gateways:
  `agent-gateway-llama-cpp` uses llama.cpp's 9931, and
  `agent-gateway-omlx` oMLX's 8000. Nothing in this proposal assumes
  one port for every gateway.

A gateway package MAY add fields of its own to the profile, such as
`agent-gateway-omlx`'s `minimumMacOS`, the oldest macOS its server
runs on. A consumer MUST ignore fields it does not know, so that a
field added later does not break an older launcher or builder.

#### `check`

```sh
agent-gateway-<name> check <model.json>…
```

Each argument is a model's `model.json` ([§2](#modeljson)), or the
model's directory. For each model, `check` writes one line, without
loading anything:

- `<id> ok` when this gateway can serve the model on this host.
- `<id> skip <code> <message>` when it cannot.

`<id>` is the model's `id` ([§2](#id)), the name `serve` lists it by.
When the `model.json` cannot be read or its `id` is unusable, the line
names the model's directory instead, with the code `model-json`.

`check` exits 0 once it has written a line for every argument, whether
those lines say `ok` or `skip`. A skip is an answer, not a failure:
what it means is the caller's to decide, and a builder fails on some
codes ([§3](#3-build)) where `serve` only warns. `check` exits
non-zero only when it cannot check at all, such as when it is called
with no arguments.

It checks what can be known before a load: that the model has files
or an endpoint and not both, the model files' format and platform
against `describe`, whether the gateway can hand the files to its
server as they are laid out, that `entry` and every file in `files[]`
exist, and their size against the host's memory. Whether the server
supports the model's architecture or quantization only shows when it
loads the model. A model with an endpoint is always skipped: it never
reaches a local server.

For the size check, `check` SHOULD compare the sum of the `size` of
the model's `weights` files with the memory the machine's accelerator
can use, not with a flat share of its RAM. On macOS that is Metal's
recommended working set (`recommendedMaxWorkingSetSize`), about 80 %
of memory on a 64 GiB Mac. A model that does not fit is skipped with
the code `memory`.

#### `serve`

```sh
agent-gateway-<name> serve [--listen <addr>] [<models dir>…] \
  [-- <server args>]
```

`serve`:

- prepares what its server takes from the models in each
  `<models dir>`, using links only and never copying model files, and
  skips with a warning every model `check` would skip. When two
  directories hold a model with the same `id`, the first one named
  wins and the other is skipped with a warning. Borrowed models in the
  Hugging Face cache are served as the server serves them today.
- with no `<models dir>`, serves the directories in
  `AGENT_STACKS_MODELS_DIR`, a colon-separated list like `PATH`, and
  without that the `share/agent-models/` of the environment it runs
  in (`$FLOX_ENV/share/agent-models` in a Flox environment). A
  service in an environment can then run `serve` with no arguments.
- starts the server, listening at `<addr>` or at
  `127.0.0.1:<defaultPort>`, and passes `<server args>` to it
  unchanged.
- serves every model it kept at that one endpoint, as
  [§1](#1-model-gateways-and-protocols) requires, and stays in the
  foreground until it receives `SIGTERM`, which it passes on to the
  server. Each model is listed and requested by its `model.json`'s
  `id`.
- writes exactly one line to standard output: `ready` once the server
  is accepting requests, or `error <code> <message>` if it cannot
  serve at all, after which it exits non-zero.
- on `SIGHUP`, once ready, SHOULD read its model directories again and
  have the running server pick up the models added and drop the ones
  removed, without a restart. When the reload fails, it warns and
  keeps serving what it served before.

Reloading is a SHOULD because the gateway is shared
([§6](#6-launch)): installing a model package into an environment
changes the directory a long-running `serve` reads, and a restart to
see it would drop every launch's loaded models and requests in flight.
Both llama.cpp's router mode and oMLX can re-read their models without
restarting, so the first two packages do it; a server that cannot
gets a restart from its service manager instead.

A stack's `bin/<stack-name>-gateway` is its gateway's `serve`, pointed
at the stack's own `share/agent-models/` with the stack's serving
parameters ([§3](#3-build)). In an environment, a service runs `serve`
instead.

For llama.cpp, the core of `serve` might look like this: a directory
of links to each GGUF file, and `llama-server` in router mode over it.

```sh
for m in "$@"; do link_gguf "$m" "$state/models"; done
llama-server --models-dir "$state/models" \
  --host "$host" --port "$port" "${server_args[@]}" >&2 &
```

#### `status`

```sh
agent-gateway-<name> status --url <url>
```

Asks a running gateway at `<url>` what it serves, and writes one JSON
object in the same shape for every server:

```json
{
  "models": [
    { "id": "qwen3-8b", "loaded": true, "contextLength": 32768 }
  ]
}
```

This is where the differences in Appendix A's last row end: each
package knows where its server reports a model's context window.

`contextLength` SHOULD be:

- for a loaded model, the window the server actually runs it with,
  which a server argument such as llama.cpp's `-c` can make smaller
  than the model was trained for;
- for a model not loaded, the context the model was trained for, or
  the window the server will enforce when it loads the model, when the
  server already knows it;
- `null` when the server reports neither.

So the value can change when a model loads, and a caller that needs
the window a request will get asks again after loading it.

#### Codes

`check` and `serve` share one set of codes:

| Code | Meaning |
| --- | --- |
| `address` | the address cannot be listened on |
| `model-json` | the `model.json` is missing, cannot be read, or breaks [§2](#modeljson): an unusable `id`, or both files and an endpoint |
| `format` | the server does not load the model files' format |
| `platform` | the server does not run on this platform |
| `layout` | the gateway cannot hand files in the model's `layout` to its server |
| `weights` | the model files are missing, do not match `files[]`, or are not what `entry` and `format` say |
| `memory` | the host does not have enough memory |
| `host` | the host doesn't meet some other requirement: an accelerator, a driver, an operating system version |
| `endpoint` | the model has an endpoint, and is not served locally |
| `other` | catch-all for errors no other code names |

`layout` is about the gateway, not the model: the same files may be
served by a gateway that takes that layout, and a gateway that learns
it needs no new model. `weights` is about the files themselves. A
gateway SHOULD use the most specific code that applies and keep
`other` for what none of them names, so that a builder can act on the
code alone ([§3](#3-build)).

#### A gateway the user already runs

`describe`, `check`, and `status` start nothing, so they work against a
server the user already runs, such as Ollama's app, as well as against
the stack's own gateway. Only `serve` starts a server.

### 8. Bindings

As a Nix function argument:

```nix
mkAgentStack {
  name = "caveman-stack";
  harness = claude-code;
  plugins = [ agent-plugin-caveman ];
  models = [ agent-model-qwen3-8b ];
  gateway = agent-gateway-llama-cpp;
}
```

The model's source lives in the model's package and the server in the
gateway package, and the stack expression only names them. With one
model, that model is also the default. With several, `modelRoles`
says which is which:

```nix
mkAgentStack {
  name = "caveman-stack";
  harness = claude-code;
  plugins = [ agent-plugin-caveman ];
  models = [ agent-model-qwen3-8b agent-model-qwen3-0_6b ];
  gateway = agent-gateway-llama-cpp;
  modelRoles = {
    default = "qwen3-8b";
    haiku = "qwen3-0.6b";
  };
}
```

As an environment manifest, where a service runs the gateway:

```toml
[install]
claude-code.pkg-path             = "claude-code"
agent-plugin-caveman.pkg-path    = "agent-stacks/agent-plugin-caveman"
agent-model-qwen3-8b.pkg-path    = "agent-stacks/agent-model-qwen3-8b"
agent-gateway-llama-cpp.pkg-path = "agent-stacks/agent-gateway-llama-cpp"

[services.gateway]
command = "agent-gateway-llama-cpp serve"
```

An externally-hosted model is installed the same way, from a package
holding only its `model.json`. That is how an organization can
publish the endpoints its stacks use.

## Open questions

1. **Context length.** The context length is fixed in
   `bin/<stack>-gateway` when the stack is built. Every model gateway
   reports a model's context window somewhere else (Appendix A), so it
   is not part of the contract in [§1](#1-model-gateways-and-protocols),
   but a gateway package's `status` reads it in one shape: the
   running window of a loaded model, the trained one otherwise
   ([§7](#status)). Should launch pass that to the agent client, and
   should a value the stack declares win over it?
2. **Lazy loading.** Resolved: a model gateway loads a model when a
   request names it ([§1](#1-model-gateways-and-protocols)), and the
   gateway is already running when a stack launches ([§6](#6-launch)).
3. **Secrets.** In the current proposal, a `model.json`'s `endpoint`
   doesn't include secrets, so if the endpoint for an externally-hosted
   model requires secrets, we have no way of letting the agent client
   authenticate. How do we let agent clients authenticate without
   leaking secrets?
4. **Single base URL.** Some agent clients, such as Claude Code, take a
   single base URL. How do they reach models at different URLs?
5. **Which protocols does an endpoint speak?** No model gateway reports
   it. When launch finds an endpoint by asking, or a source gives no
   protocols, launch has to recognise the gateway by its own endpoints
   (llama.cpp's `/props`, Ollama's `/api/ps`, oMLX's
   `/v1/models/status`) or make a test call per protocol. A gateway
   package's `describe` answers this for a gateway launch knows
   ([§7](#7-gateway-packages)). For one it does not, such as a remote
   endpoint with no `protocols`, is that fallback part of this proposal,
   or left to each launcher?
6. **`model.json` and the efforts it borrows from.** Resolved in
   part: [§2](#modeljson) specifies `model.json`,
   `agent-gateway-llama-cpp` and `agent-gateway-omlx` feed their
   servers from it with links alone, and `endpoint.json` is
   its `endpoint` object, fields unchanged. What stays open is how it
   tracks three efforts it borrows from:
   - CNCF ModelPack PR #202, a typed `config`, which may give
     `quantization` and `capabilities` shapes of their own.
   - CNCF ModelPack PR #240, state artifacts.
   - Ollama's per-runner manifest list, which records which of its
     runners loads which files: prior art for the `ollama` layout and
     for choosing a gateway by format.

   When these land, should `model.json` follow them under a new
   `schemaVersion`, or keep its own shapes and map between them?
7. **Naming a borrowed model.** A borrowed model is declared by its
   repository, but launch sees the ID its model gateway lists, and
   gateways derive that ID differently. Is the repository enough for a
   launcher to find the model and announce it, or does the declaration
   also need the ID?
8. **Should a launcher ever start the gateway?** This proposal says
   never: the user or a service manager runs `bin/<stack>-gateway`,
   and the launcher connects or names the command. An earlier draft
   had the launcher start, share, and stop the servers itself, which
   is more convenient for one person on one machine but needs a
   watcher that outlives the agent client, amending §7. Is that worth
   keeping as an option?
9. **One loopback port through a sandbox.** Both sandboxes in Proposal
   0001 filter by host name through a proxy. Can each allow
   `127.0.0.1:<port>` and nothing else on loopback, or does a local
   gateway open the whole interface?
10. **Serving parameters per model.** `serve` takes one set of server
    arguments for all the models it serves, but a context length or
    an offload setting is often per model. Does that belong in
    `model.json`, in the gateway's own per-model configuration (such
    as a llama.cpp preset file), or in both?

## Appendix A: Model gateways that meet the contract

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
