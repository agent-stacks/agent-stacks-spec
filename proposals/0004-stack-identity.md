# Proposal 0004: Stack identity and telemetry

- Status: Draft
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.1, and the sections it
  would touch: §2, §5.1, §6, §7, §9, Appendix A
- Scope: how a stack says what it is, on disk and while it runs, and
  how an operator requires telemetry from every stack on a machine.

## Summary

A stack carries no record of what it is, and nothing it runs says
which stack it came from. Tools that govern, secure, or observe agents
have to guess. This proposal adds three things:

1. **Record.** A built stack MUST carry a record at
   `share/agent-stacks/<stack-name>.json`, written by the builder,
   naming the specification version, the stack, its launcher, its
   agent client, and its plugins.
2. **Runtime identity.** Before it runs the agent client, the launcher
   puts the stack's identity into the agent client's environment and
   its OpenTelemetry resource attributes.
3. **Telemetry policy.** An operator MAY require telemetry from every
   stack on a machine. The launcher translates that policy into each
   agent client's own settings, and refuses to launch an agent client
   it cannot configure.

The record is the first requirement this specification defines of its
own (§2). Everything else here is SHOULD or MAY.

## Motivation

### Something now reads it

§11.1 deferred a stack manifest until something would read it: "a
format specified before it has a consumer is specified from
imagination." Three kinds of reader exist now, and each is stuck
without one.

- **Discovery tools** inventory the agents on developer machines for
  security and governance teams. They look for known client paths
  under the home directory: `~/.claude`, `~/.local/bin/claude`, an npm
  package, an editor extension. A carried agent client runs from the
  stack's dependency set, for example
  `/nix/store/<hash>-claude-code-<version>/bin/claude`. None of those
  paths match it, and nothing at the stack root says what it is. These
  tools either miss a stack or guess from the shape of a directory, and
  each guesses differently.
- **Observability backends** key traces and metrics on OpenTelemetry
  resource attributes, usually `service.name`. An agent client sets its
  own (`claude-code`, `codex-cli`), so a trace says which agent ran,
  but not which stack, which version of it, or which plugins it
  carried.
- **Gateways and allowlists** match MCP servers by exact command,
  arguments, or URL. A store path changes on every rebuild, so they
  need a stable name beside the digest.

### Why not a `stack.json` at the root

§11.1 sketched a `stack.json` at the stack root. Stacks are installed
side by side (§5.4), and with Nix or Flox that usually means several
stacks merged into one profile or environment. A fixed name at the
root collides on the second stack installed.

A record named for its stack follows the pattern the layout already
uses for plugins (`share/agent-plugins/<name>/`) and that proposal 0001
uses for sandboxes (`share/agent-sandboxes/<name>/`). After a merge it
is also the only record of which plugins belong to which stack.

### Why a record is not a second source of truth

§11.1's other objection is that the builder already knows what it
made, so a file restating it would be a second source of truth. Here
the builder writes the record from the declaration, as a lockfile
records a resolution. Nobody edits it, and in a content-addressed store
nobody can. The declaration stays the only thing an author writes.

### Why runtime identity as well

A record covers a stack on disk. It does not cover a stack assembled
for one launch (`agent-stacks launch claude`, §6.5), which has no
root, or a trace that reaches a collector with no file attached. The
launcher is the one step every binding passes through, so it is where
identity is stamped.

### Why the operator owns telemetry

Requiring telemetry is an organization's decision. A stack's author
cannot know where a given organization sends its telemetry, or whether
it allows prompt text to leave the machine. Agent clients expose
telemetry in different ways (environment variables, a config file, a
vendor's admin console), and they disagree on defaults: one logs
prompts unless told not to. The launcher already adapts the neutral
layout to each agent client at launch (§7), so it is the natural place
to adapt one policy to each agent client as well.

## Out of scope

- **Signing.** A file cannot carry its own digest. Signing a stack
  with a detached statement is left for later; see open question 2.
- **A bill of materials.** The record can point at one. It does not
  define BOM semantics, and CycloneDX already does.
- **Span conventions.** Which span attributes an agent client emits
  (OTel GenAI semantic conventions, OpenInference, its own names) is
  the agent client's business. Collectors translate between them.
- **Enforcement inside the agent.** Blocking tool calls, kill switches,
  and guardrails belong to a sandbox (proposal 0001), a gateway, or the
  agent client's own hooks.
- **Agent registries and cards.** A2A Agent Cards, MCP Registry
  entries, and OASF records describe agents published to a directory
  or reachable over a network. A record can be exported to them; it
  does not replace them.

## Proposal

### 1. The stack record

#### Layout

1. A stack produced by a builder MUST carry exactly one record per
   launcher, at `<stack root>/share/agent-stacks/<stack-name>.json`,
   where `<stack-name>` is the launcher's basename (§5.1(4)).
2. A directory is a stack root when it holds
   `share/agent-stacks/<name>.json` and `bin/<name>`. A directory with
   several records holds several stacks, as a merged profile does.
3. A tool SHOULD identify a stack by its record, and SHOULD NOT infer
   one from the shape of a directory when a record is present.

```text
caveman-stack/
├── bin/
│   └── caveman-stack
└── share/
    ├── agent-plugins/
    │   └── caveman/...
    └── agent-stacks/
        ├── caveman-stack.json         the record
        └── caveman-stack.cdx.json     optional bill of materials
```

#### Fields

The record is a JSON object with a closed schema, published at
`https://agent-stacks.org/schemas/<version>/stack.schema.json`. Its
shape follows the Agent Plugins manifest (§5): `$schema` versioning,
the same name constraints (§5.5), and a reverse-domain `extensions`
object (§8.1).

Required:

- `$schema`: the schema URI. Its version is the version of this
  specification the stack was built against.
- `name`: the stack's name, under the Agent Plugins name constraints.
- `launcher`: the launcher's path relative to the stack root,
  `bin/<name>`.
- `agentClient`: an object with `name`, `provenance` (`carried` or
  `borrowed`, §6.2), and `version` when carried.

Recommended:

- `version`: the stack's own version.
- `plugins`: one entry per plugin, with `name`, `version` (from its
  `plugin.json`, when it has one), `path` relative to the stack root,
  and `source` (the URL and revision §6.3 pins).

Optional:

- `description`: one line for a human.
- `sandbox`: the name of the sandbox under `share/agent-sandboxes/`
  (proposal 0001).
- `bom`: a path relative to the stack root of a bill of materials,
  such as a CycloneDX document, which carries per-component hashes and
  the MCP servers each plugin declares.
- `extensions`: data keyed by reverse-domain namespace, processed as
  Agent Plugins §8.1 processes manifest extensions. A discovery or
  governance tool MAY read its own namespace, such as an owner or a
  risk tier an organization adds when it repackages a stack.

A reader SHOULD report and ignore a field it does not know, so that a
newer record still reads.

```json
{
  "$schema": "https://agent-stacks.org/schemas/0.2.0/stack.schema.json",
  "name": "caveman-stack",
  "version": "1.4.0",
  "launcher": "bin/caveman-stack",
  "agentClient": {
    "name": "claude-code",
    "version": "2.1.310",
    "provenance": "carried"
  },
  "plugins": [
    {
      "name": "caveman",
      "version": "2.7.0",
      "path": "share/agent-plugins/caveman",
      "source": { "url": "https://github.com/example/caveman", "rev": "3f2a9c1" }
    }
  ],
  "sandbox": "nono",
  "bom": "share/agent-stacks/caveman-stack.cdx.json"
}
```

The record carries no digest of the stack. A stack's content digest is
a property of its dependency set (in the reference implementation, its
store path), and a file cannot contain its own.

#### Build

1. A builder SHOULD write the record from the declaration (§6). An
   author SHOULD NOT write one by hand.
2. A builder SHOULD validate the record against its schema and fail
   the build rather than emit one that does not validate, as §10(3)
   asks of the plugin tree.
3. A builder MAY write a bill of materials beside the record and name
   it in `bom`.

### 2. Runtime identity

§7 lists four steps. This proposal adds one between resolving the
agent and executing it. When a stack also carries secrets (proposal
0003) or a sandbox (proposal 0001), this step runs first, so that every
process they start inherits the identity.

**Identify.**

1. If the stack has no record, because it was assembled for this
   launch, write one into the per-launch staging directory.
2. Set `AGENT_STACK_RECORD` to the record's absolute path, so that
   hooks and plugins running inside the agent client can read it.
3. Append to `OTEL_RESOURCE_ATTRIBUTES`, never overwriting a key
   already set:
   - `service.namespace=<stack name>`
   - `agent_stack.name=<stack name>`
   - `agent_stack.version=<version>`, when the record has one
   - `agent_stack.spec_version=<version from $schema>`
   - `agent_stack.provenance=<carried or borrowed>`

A launcher SHOULD NOT set `service.name`. The agent client names
itself, so a backend sees which agent client ran inside which stack.

Every value above is restricted to the name constraints, and a value
outside them is left out rather than escaped, because some agent
clients drop resource attributes whose values contain spaces.

`service.namespace` is a stable OpenTelemetry attribute. The OTel GenAI
semantic conventions define agent attributes only on spans, and are
still in development, so the `agent_stack.*` names are proposed here
and offered upstream if this proposal is accepted (open question 3).

### 3. Telemetry policy

#### Where it lives

1. A launcher SHOULD read a telemetry policy from a system location,
   `/etc/agent-stacks/policy.json` on Linux and
   `/Library/Application Support/agent-stacks/policy.json` on macOS,
   and from a user location, `$XDG_CONFIG_HOME/agent-stacks/policy.json`.
   Where both set a field, the system file wins, because it is the one
   an operator deploys, typically through device management.
2. A stack SHOULD NOT carry a telemetry policy, and a launcher SHOULD
   ignore one that it does.

#### What it says

```json
{
  "telemetry": {
    "required": true,
    "endpoint": "https://otel.example.com",
    "protocol": "http/protobuf",
    "signals": ["traces", "metrics", "logs"],
    "headers": { "Authorization": { "secret": "OTEL_AUTHORIZATION" } },
    "capture": { "prompts": false, "toolDetails": false }
  }
}
```

- `required`: whether a stack may run without exporting telemetry.
- `endpoint`, `protocol`: an OTLP endpoint and protocol (`grpc` or
  `http/protobuf`).
- `signals`: which of traces, metrics, and logs to export.
- `headers`: header values given as references to secrets, resolved
  the way proposal 0003 resolves a plugin's secrets. A policy SHOULD
  NOT hold a header value.
- `capture`: whether prompt text and tool arguments may be exported.
  Every class defaults to `false`.

#### Launch

§7 gains a second step after **Identify**.

**Configure telemetry.** When a policy exists:

1. Translate it into the agent client's own settings for this launch
   only, through its environment or a config file in the staging
   directory. Never edit the consumer's agent configuration (§9(1)).
2. Set every capture setting the agent client has explicitly, to off
   where the policy says off, rather than relying on its defaults.
3. When `required` is `true` and the agent client cannot be configured
   to export (it is unknown to the launcher, or its export is set only
   in its vendor's admin console), refuse to launch, naming the agent
   client and the policy file.

When no policy exists, change nothing.

The per-agent-client work belongs in a small adapter, as it does for
sandboxes in proposal 0001. Appendix A lists what the adapters for
common agent clients set today.

### 4. Conformance

§2 says this document defines no requirements of its own. This
proposal changes that to one:

> A stack produced by a builder for this specification at 0.2.0 or
> later MUST carry one stack record per launcher (§5.1).

The record is what names the version a stack claims, so every other
claim a stack makes rests on it. A stack assembled for one launch has
no builder, and gets its record from the launcher instead.

Appendix A gains three lines:

- carries a generated record per launcher at
  `share/agent-stacks/<name>.json`
- stamps the stack's identity on the agent client's environment at
  launch
- applies the operator's telemetry policy, refusing to launch when it
  is required and cannot be applied

### 5. Bindings

This subsection is non-normative.

As a Nix function, nothing changes in the expression. `mkAgentStack`
already has every field the record needs, and writes
`$out/share/agent-stacks/${name}.json` from its arguments.

As an environment manifest, the environment is the declaration (§6.5).
A stack package installed into it brings its own record. Where the
environment itself composes the agent client and plugins, the tool
that builds the environment can write the record when it builds; until
it does, the launcher writes one per launch, as for a stack assembled
from the command line.

As a command line, `agent-stacks launch claude` writes a record into
its staging directory, with `provenance` taken from how it resolved
the agent client.

## Considered alternatives

**A record at a fixed root path** (`stack.json`, or a `.agent-stacks/`
directory). Most visible at the top of a stack, and rejected because
the second stack merged into a profile collides with the first.

**Recognizing a stack by its shape**, a directory with `bin/` and
`share/agent-plugins/`. It works for stacks that exist today, but every
tool guesses differently, and the shape cannot say which specification
version, which agent client, or whether that agent client is carried
or borrowed.

**An existing format as the record.** An A2A Agent Card describes an
agent reachable over a network, and its well-known path implies a
server. CycloneDX and SPDX describe what a stack contains, not what to
run or how. OASF records belong to a directory. Each is a better
export or companion than a record.

**Requiring a span convention**, such as OpenInference or the OTel GenAI
conventions. This specification cannot make an agent client emit one.
It can configure where telemetry goes and which resource attributes it
carries, and backends already convert between conventions.

**Telemetry declared by the stack.** It puts the destination in the
hands of an author who cannot know it, and a stack that exports
prompts somewhere is something a consumer would have to inspect the
stack to find out.

## Rollout

1. **Record and identity.** The reference implementation writes the
   record and stamps identity, marked experimental, so that discovery
   tools and backends can read real stacks and report back on the
   fields.
2. **Telemetry policy for two agent clients.** Claude Code and Codex,
   which configure export in different ways (environment and config
   file), as a test of whether the adapter boundary is in the right
   place.
3. **Into the specification.** The record and its requirement in
   0.2.0, with the open questions below answered or restated.

## Open questions

1. **Finding records.** Installed stacks are reachable through
   home-directory links into the store, such as
   `~/.nix-profile/share/agent-stacks/`. A stack run straight from a
   store path, or assembled for one launch, is not. Should the launcher
   append one line per launch to
   `$XDG_STATE_HOME/agent-stacks/launches.jsonl` (time, stack name,
   record path, agent client path, provenance), so that a discovery tool
   has one fixed file to read? That needs a carve-out from §7, which
   says the launcher writes nothing, and a rule for how long lines are
   kept.
2. **Signing.** A detached signature over the stack directory (for
   example OpenSSF Model Signing, which signs an in-toto statement
   listing every file's digest), the store's own signatures, or both?
3. **Attribute names.** Keep `agent_stack.*` here, or propose agent
   resource attributes to the OTel GenAI semantic conventions first?
4. **Author and operator.** Can an author mark a stack as requiring
   telemetry, for a platform team's stack, or is that only ever the
   operator's call?
5. **Nested stacks (§11.2).** Does an outer stack's record name its
   inner stacks? When an outer launcher starts an inner one, does the
   inner identity append to the outer's attributes or replace them?
6. **Launch defaults.** Should the record carry a default command or
   arguments for the launcher?
7. **Agent clients' own managed settings.** Some agent clients read an
   organization's settings file of their own (Claude Code's managed
   settings, for one). When both set telemetry, which wins?
8. **Namespace.** The schema URI and the `agent_stack` prefix assume
   the project's domain, agent-stacks.org, as proposal 0003 does for
   `org.agent-stacks`.

## Appendix A: Agent client telemetry settings

This appendix is non-normative. It records what an adapter would set
for each agent client as of October 2026, from each agent client's own
documentation. It belongs to adapters, not to this specification, and
will go stale.

| Agent client | How export is turned on | Prompt text by default | Launcher can configure it |
|---|---|---|---|
| Claude Code | `CLAUDE_CODE_ENABLE_TELEMETRY=1` and the standard `OTEL_*` variables; traces are in beta | Off | Yes, through the environment |
| Codex CLI | The `[otel]` table in its config file | Off | Yes, through a staged config file |
| Gemini CLI | `telemetry.*` settings or `GEMINI_TELEMETRY_*` variables | On | Yes; the adapter has to turn prompt logging off |
| Cursor | Configured by an administrator in Cursor's own console | Off | No |

Sources:

- Claude Code: <https://code.claude.com/docs/en/monitoring-usage>
- Codex CLI: <https://learn.chatgpt.com/docs/config-file/config-advanced>
- Gemini CLI: <https://geminicli.com/docs/cli/telemetry/>
- Cursor: <https://cursor.com/docs/enterprise/opentelemetry-export>
