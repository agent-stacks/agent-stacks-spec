# Proposal 0004: Stack identity, discovery, and telemetry

- Status: Draft
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.1, and the sections it
  would touch: §2, §5.1, §6, §7, §9, Appendix A
- Scope: how a stack says what it is, on disk and while it runs, how a
  tool finds the stacks on a machine, and how an operator requires
  telemetry from every stack on a machine.

## Summary

A stack carries no record of what it is, nothing it runs says which
stack it came from, and nothing tells a tool where to look for one.
Tools that govern, secure, or observe agents have to guess. This
proposal adds four things:

1. **Record.** A built stack MUST carry a record at
   `share/agent-stacks/<stack-name>.json`, written by the builder,
   naming the specification version, the stack, its launcher, its
   agent client, and its plugins.
2. **Discovery.** A tool finds records under the `share/` directories
   that stacks are installed into, and the launcher registers each
   launch in one directory per user, so that stacks with no fixed
   location (a project's environment, a stack assembled for one launch)
   can be found too. Everything a tool needs is read from files. It
   never has to run anything in the stack.
3. **Runtime identity.** Before it runs the agent client, the launcher
   puts the stack's identity into the agent client's environment and
   its OpenTelemetry resource attributes.
4. **Telemetry policy.** An operator MAY require telemetry from every
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

  The same tools inventory MCP servers and plugins by reading each
  agent client's own configuration (`~/.claude.json`,
  `~/.codex/config.toml`, `~/.cursor/mcp.json`). A stack is built not
  to touch that configuration (§9(1)). Its plugins are staged per
  launch and handed to the agent client for that one process, so its
  MCP servers never appear where these tools read. The hygiene this
  specification asks for is what makes a stack invisible to them.
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

### Why a record also needs a way to be found

A record says what a stack is once a tool has found it. Stacks live
wherever their binding puts them: a store path, a Nix profile, a
Homebrew prefix, a project's `.flox/` directory. Placing the record
under `share/` means a stack installed into any prefix brings its
record to `<prefix>/share/agent-stacks/`, the way a desktop entry
lands in `<prefix>/share/applications/`. A tool that knows the
prefixes finds the installed stacks without walking the disk.

Two kinds of stack have no fixed prefix. A project environment can be
anywhere on disk, and a stack assembled for one launch has no root
until it runs. The launcher is the one place both pass through, so it
is where a stack registers that it ran.

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
- **Finding agents that hide.** Records and registrations are written
  by tools that follow this specification. Someone who wants to run an
  agent unseen can run it without either. Discovery here is
  cooperative. Catching an agent that avoids it is the job of endpoint
  monitoring, which watches processes rather than files.

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
  and `source` (the URL, revision, and content digest §6.3 pins).

Optional:

- `source`: where the stack's declaration came from, as a `url` and
  `rev`, when the builder knows it (a flake reference does; a local
  path does not).
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
  "source": { "url": "https://github.com/example/stacks", "rev": "9e41b07" },
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
      "source": {
        "url": "https://github.com/example/caveman",
        "rev": "3f2a9c1",
        "digest": "sha256-ekpf8REO3/LrEJ7wCQPF20qsHj7qqkKZnF9Jv2FV/0I="
      }
    }
  ],
  "sandbox": "nono",
  "bom": "share/agent-stacks/caveman-stack.cdx.json"
}
```

The record carries no digest of the stack. A stack's content digest is
a property of its dependency set (in the reference implementation, its
store path), and a file cannot contain its own.

A stack's name is not unique. Two authors can both build a
`dev-stack`, and renaming a stack changes nothing else about it. A tool
that keys an allow or deny rule on a stack SHOULD key it on the stack's
`source` and its plugins' digests, or on its store path, and not on
its name alone.

The record names each plugin and where it is. It does not list a
plugin's skills or MCP servers, because the plugin's own files already
say that (§6.4). [Reading a stack](#reading-a-stack) is how a tool gets
them.

#### Build

1. A builder SHOULD write the record from the declaration (§6). An
   author SHOULD NOT write one by hand.
2. A builder SHOULD validate the record against its schema and fail
   the build rather than emit one that does not validate, as §10(3)
   asks of the plugin tree.
3. A builder MAY write a bill of materials beside the record and name
   it in `bom`.

### 2. Discovery

In this part, a *detector* is any tool that looks for the stacks on a
machine: an inventory agent, a governance scanner, or a `list` command
in an agent stack client. A detector finds stacks in three ways, and
learns what they are by reading files.

#### Where records are

1. A detector SHOULD look for `agent-stacks/*.json` under each `share/`
   directory that stacks are installed into: the system and per-user
   prefixes of the package managers in use, and each user's profiles.
   [Appendix B](#appendix-b-where-installed-stacks-land) lists the ones
   known today. The list will grow, which is why it is an appendix.
2. A detector MAY also look in a content-addressed store
   (`/nix/store/*/share/agent-stacks/*.json`). That finds stacks that
   are built but not installed, and old generations of installed ones.
3. A detector SHOULD resolve links and report each record once, by its
   resolved path. One stack in a store is reachable from every profile
   and environment it is installed into.

A detector SHOULD say how it found each stack, because the three ways
mean different things:

- **Present**: its record is on disk, but only in a store.
- **Installed**: its record is reachable from a prefix or profile that
  a user's environment uses.
- **Launched**: a registration says it ran, and when.

A stack can be installed and launched at once.

#### Registering a launch

1. On every launch, the launcher SHOULD write a registration to
   `$XDG_STATE_HOME/agent-stacks/launched/`, or to
   `~/.local/state/agent-stacks/launched/` when `XDG_STATE_HOME` is
   unset or relative. This holds on macOS as well as Linux, so a
   detector has one path to check per user. The reference
   implementation already keeps its own state under
   `$XDG_STATE_HOME/agent-stacks/`.
2. The launcher writes one file per stack location: the record's path,
   or for a stack assembled for one launch, the plugin directory it read
   together with the agent client it ran. Launching the same stack again
   replaces its registration, so the directory grows with the number of
   stacks a user runs, not the number of launches. The file name is the
   launcher's choice, and a detector reads every `*.json` in the
   directory.
3. The launcher writes the file atomically (to a temporary file in the
   same directory, then renamed into place), so a detector never reads
   half of one.
4. A launcher that cannot write a registration SHOULD warn and launch
   anyway.

A registration holds:

- `$schema`: the registration schema URI, versioned with the record's.
- `recordPath`: the record's absolute path as the launcher found it,
  without resolving links, so that a project environment's path still
  names its project.
- `record`: for a stack assembled for one launch, the record itself.
  Its staged copy is removed with the staging directory, so the
  registration keeps it. A registration has `recordPath` or `record`,
  never both.
- `agentClient`: the absolute path of the agent client executable this
  launch ran, with links resolved. That is the path a process monitor
  sees.
- `provenance`: `carried` or `borrowed`, as resolved at this launch. It
  can differ from the record's. A stack built to carry its agent client
  can still find another one first on `PATH` (§9(3)), and a detector
  that compares the two sees it.
- `launched`: the time of the most recent launch, in RFC 3339 UTC.

It holds nothing else. Arguments to the agent client can carry a
prompt and its environment can carry secrets, and a registration is
readable by anything running as its user, so neither goes in.

```json
{
  "$schema": "https://agent-stacks.org/schemas/0.2.0/registration.schema.json",
  "recordPath": "/home/alex/src/app/.flox/run/x86_64-linux.app.dev/share/agent-stacks/caveman-stack.json",
  "agentClient": "/nix/store/<hash>-claude-code-2.1.310/bin/claude",
  "provenance": "carried",
  "launched": "2026-11-09T16:41:27Z"
}
```

A launcher MAY remove a registration whose record no longer exists,
and SHOULD NOT remove any other. A detector SHOULD report such a
registration as stale rather than as a stack. A project environment
keeps its path when it is rebuilt, so its registration stays current.
A stack run straight from a store path leaves a stale registration
behind once the store is collected.

Registrations are per user. A detector that runs as an administrator
reads each user's directory in turn, and each is writable only by its
owner.

Writing a registration needs no change to §7. §7 keeps the launcher
out of two places, the stack and the consumer's agent configuration,
and a registration is in neither.

#### Reading a stack

A detector learns what a stack is by reading files. It SHOULD NOT run
the launcher, the agent client, or anything else in the stack to find
out.

1. Read the record. It names the stack and its source, the agent
   client and how it is obtained, and each plugin with its source,
   digest, and path.
2. For each plugin, read `plugin.json` and `mcp.json` at the plugin's
   path, and list its `skills/`. Agent Plugins fixes where each of
   these is. Together they say which MCP servers the agent client will
   start or connect to, and which skills it will load.
3. If the record names a `bom`, read it for package identifiers and
   per-component digests.
4. If the record resolves into a content-addressed store, the store
   path is the stack's digest. Two records that resolve to the same
   store path are the same stack.

What a detector reads is what the stack declares, and a detector
SHOULD NOT present it as more than that. Three things can differ at
launch:

- The launcher drops an invalid component and keeps going (§7(1)), so
  a declared server can fail to load.
- A borrowed agent client also loads whatever its own configuration
  names. A detector that already reads that configuration sees both
  halves.
- A value in `mcp.json` can be a `${NAME}` placeholder filled in at
  launch (proposal 0003). A detector sees the name, never the value.

If a detector needs something it can only learn by running part of a
stack, that is a gap in this proposal.

A detector that sees a running agent client attributes it to a stack
through the `AGENT_STACK_RECORD` in its environment
([Runtime identity](#3-runtime-identity)). Registrations say that a
stack ran, not which process is running it.

#### What a detector can trust

- A record in a content-addressed store says what its builder wrote.
  The store is immutable, and the path names the content.
- A record anywhere else, and every registration, says what its
  owner's tools wrote. Its owner can change either.
- A registration is a pointer, not evidence. A detector SHOULD read the
  record it points to, and treat what the registration adds (the agent
  client's path, the provenance, the time) as what the launcher
  reported. An assembled stack's `record` has nothing else to point to.
- Absence proves nothing. A stack never launched by a launcher that
  follows this proposal leaves no registration, and a user can delete
  their own.

Signing, which would let a record outside a store say who built it,
is open question 2.

### 3. Runtime identity

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
4. Register the launch, as [Registering a launch](#registering-a-launch)
   describes.

A launcher SHOULD NOT set `service.name`. The agent client names
itself, so a backend sees which agent client ran inside which stack.

Every value above is restricted to the name constraints, and a value
outside them is left out rather than escaped, because some agent
clients drop resource attributes whose values contain spaces.

`service.namespace` is a stable OpenTelemetry attribute. The OTel GenAI
semantic conventions define agent attributes only on spans, and are
still in development, so the `agent_stack.*` names are proposed here
and offered upstream if this proposal is accepted (open question 3).

### 4. Telemetry policy

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

### 5. Conformance

§2 says this document defines no requirements of its own. This
proposal changes that to one:

> A stack produced by a builder for this specification at 0.2.0 or
> later MUST carry one stack record per launcher (§5.1).

The record is what names the version a stack claims, so every other
claim a stack makes rests on it. A stack assembled for one launch has
no builder, and gets its record from the launcher instead.

The specification's Appendix A gains four lines:

- carries a generated record per launcher at
  `share/agent-stacks/<name>.json`
- registers each launch where a detector can find it, holding no
  prompt, argument, or secret
- stamps the stack's identity on the agent client's environment at
  launch
- applies the operator's telemetry policy, refusing to launch when it
  is required and cannot be applied

### 6. Bindings

This subsection is non-normative.

As a Nix function, nothing changes in the expression. `mkAgentStack`
already has every field the record needs, and writes
`$out/share/agent-stacks/${name}.json` from its arguments. Installed
with `nix profile`, the record shows up in
`~/.nix-profile/share/agent-stacks/`, where a detector already looks.

As an environment manifest, the environment is the declaration (§6.5).
A stack package installed into it brings its own record. Where the
environment itself composes the agent client and plugins, the tool
that builds the environment can write the record when it builds; until
it does, the launcher writes one per launch, as for a stack assembled
from the command line. A project environment lives in its project, so
its record is under that project's `.flox/run/`, which no fixed list
of prefixes reaches. Registration is how a detector finds it.

As a command line, `agent-stacks launch claude` writes a record into
its staging directory, with `provenance` taken from how it resolved
the agent client, and puts that record inline in its registration.

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

**The plugins' MCP servers and skills restated in the record.** A
detector would read one file instead of a few. Rejected because the
plugin's own files already say it (§6.4), a builder that restates them
can get them wrong, and the files are at fixed locations a detector can
read without running anything. A BOM can still carry a copy for tools
that want one.

**An append-only launch log** (`launches.jsonl`, one line per launch).
It grows without bound, needs a rule for how long lines are kept, and
a detector has to read all of it to learn which stacks exist. One file
per stack location is bounded by the number of stacks a user runs,
needs no rotation, and is read with a directory listing.

**A registry shared by all users** (`/var/lib/agent-stacks/`). Every
user would need to write to it, which lets any user forge or delete
another's entries, or a privileged helper would have to write it for
them. A directory per user is readable by an administrator's detector
and writable only by its owner.

**Writing into the agent client's own configuration**, so that
existing detectors see a stack's MCP servers where they already look.
It breaks §9(1), and the stack's plugins would load in every session
of that agent client rather than only the stack's.

**Walking the filesystem** for `share/agent-stacks/`. It finds project
environments without any help from the launcher, at the cost of
reading the whole disk on every scan. A detector MAY do it, but
discovery should not depend on it.

**Requiring a span convention**, such as OpenInference or the OTel GenAI
conventions. This specification cannot make an agent client emit one.
It can configure where telemetry goes and which resource attributes it
carries, and backends already convert between conventions.

**Telemetry declared by the stack.** It puts the destination in the
hands of an author who cannot know it, and a stack that exports
prompts somewhere is something a consumer would have to inspect the
stack to find out.

## Rollout

1. **Record, discovery, and identity.** The reference implementation
   writes the record and registrations and stamps identity, marked
   experimental, so that discovery tools and backends can read real
   stacks and report back on the fields. A detector in the reference
   implementation that lists the stacks on a machine is the first test
   of whether [Discovery](#2-discovery) is enough.
2. **Telemetry policy for two agent clients.** Claude Code and Codex,
   which configure export in different ways (environment and config
   file), as a test of whether the adapter boundary is in the right
   place.
3. **Into the specification.** The record and its requirement in
   0.2.0, with the open questions below answered or restated.

## Open questions

1. **Registrations a detector cannot see.** A launcher honors an
   absolute `XDG_STATE_HOME`, and a detector running as another user
   cannot read that user's environment, so it checks the default path
   and misses registrations written elsewhere. Should registrations
   always go to `~/.local/state/agent-stacks/launched/`, whatever
   `XDG_STATE_HOME` says? And can an operator's policy make a failed
   registration refuse the launch, as `telemetry.required` does for
   telemetry?
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
9. **Agent client names.** `agentClient.name` has no shared vocabulary,
   and a detector that matches on it needs one. Should this
   specification keep a non-normative list of names for common agent
   clients, or should the record carry a package URL (purl) beside the
   name?

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

## Appendix B: Where installed stacks land

This appendix is non-normative. It lists where a stack's record ends
up when it is installed the ways known as of November 2026, so that a
detector knows where to look. It will grow.

| Installed with | Where the record lands |
|---|---|
| `nix profile` | `~/.nix-profile/share/agent-stacks/`, or `$XDG_STATE_HOME/nix/profile/share/agent-stacks/` when Nix's `use-xdg-base-directories` is set |
| The default Nix profile | `/nix/var/nix/profiles/default/share/agent-stacks/` |
| NixOS or nix-darwin system packages | `/run/current-system/sw/share/agent-stacks/` |
| Home Manager as a NixOS or nix-darwin module | `/etc/profiles/per-user/<user>/share/agent-stacks/` |
| A Flox environment | `<project>/.flox/run/<system>.<env>.<mode>/share/agent-stacks/`, wherever the project is. Found through registrations. |
| Homebrew | `/opt/homebrew/share/agent-stacks/` on Apple silicon, `/usr/local/share/agent-stacks/` on Intel macOS, `/home/linuxbrew/.linuxbrew/share/agent-stacks/` on Linux |
| Distribution packages (apt, dnf) | `/usr/share/agent-stacks/` |
| A local install (`make install` and similar) | `/usr/local/share/agent-stacks/`, or `~/.local/share/agent-stacks/` for one user |
| Built but not installed | `/nix/store/*/share/agent-stacks/` |
| Assembled for one launch | Nowhere on disk once the launch ends. Found through registrations, which carry the record inline. |
