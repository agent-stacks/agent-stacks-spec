# Proposals to the composed specifications

This document is non-normative. It accompanies the
[Agent Stacks specification](spec/0.1.0.md).

Each item below is something the reference implementation does that
the composed specifications do not describe, or in one case forbid.
They are written as proposals to those specifications, not as rules
for stacks, and none has been submitted upstream yet. Each states what
we do, why, what it costs, and where it belongs if accepted.

## 1. Dependencies resolved at build time

Target: Agent Plugins.

What: Rewrite the runtime references in a plugin's format-fixed
positions to point at interpreters carried in the package's own
dependency set, reached through a plugin-local `bin/` of links, so
that `${CLAUDE_PLUGIN_ROOT}/bin/<name>` is a real path.

Why: Agent Plugins describes the package. It does not say where the
`python3` in a skill's shebang comes from, so in practice it comes
from the consumer's `PATH`, and a plugin that works on the author's
machine fails or silently misbehaves on the consumer's. Resolving at
build time closes that hole without asking anything of the consumer.

Cost: Every runtime a plugin names becomes a build input of that
plugin, so the mapping table stays lean by necessity. A plugin's
dependency set grows to include its interpreters, which is the
trade being made deliberately.

Where it belongs: A section in Agent Plugins describing
build-time-resolved packages as a recognized way to build a plugin,
leaving the resolution mechanism to the builder.

## 2. Rewrite only where the format fixes the meaning

Target: Agent Plugins.

What: Rewrite exactly two positions: a script's shebang, and a bare
`command` in `mcp.json`. Everything else — runtime names in script
bodies, in `SKILL.md` prose, in examples — is left untouched and must
be declared explicitly by the package if it matters.

Why: Those two positions are unambiguous by format: a shebang is an
interpreter reference and nothing else; an `mcp.json` `command` is a
binary to execute and nothing else. Prose is not. A rewrite that
catches `python3 -m scripts` in a script body also catches it in the
documentation next to it, and a package whose contents no longer say
what its author wrote is worse than one that fails loudly.

Cost: Body text and `SKILL.md` need per-package attention. The pass
will not guess, so some plugins need explicit substitutions written by
hand.

Where it belongs: A rule in Agent Plugins about where a builder may
rewrite, not about how much.

## 3. Binaries a skill's markdown expects

Target: Agent Skills, with a discovery rule in Agent Plugins.

What: Let a plugin declare the binaries its markdown tells the agent
to run — `rg`, `jq`, and the like — as a list. Packaging resolves the
list; the agent client decides what to do with it.

Why: A `SKILL.md` that instructs the agent to run `rg` has a real
dependency that the format cannot express. Substitution must not guess
at it, because prose is not a fixed position (proposal 2), so the only way
for it to be known is for the author to say so.

Cost: A new frontmatter field, and the question of what an agent client
does when a declared binary is absent: refuse the skill, warn, or let
the agent discover the failure. This proposal deliberately leaves that
policy open.

Where it belongs: The declaration belongs in Agent Skills
frontmatter, since it is a property of the skill. Any rule about how a
plugin aggregates its skills' declarations belongs in Agent Plugins.

## 4. Path containment and hash-addressed stores

Target: Agent Plugins §4.1(3), in both 1.0.0 and 1.1.0.

What: Amend clause 3 so that a symlink inside the plugin root may
resolve to a target outside it when the target lies within a
hash-addressed store root:

```text
A symlink within the plugin root MAY resolve to a target outside the
plugin root when that target lies within a hash-addressed store
root. A hash-addressed store root is a filesystem location whose
entries are named by a cryptographic digest and are immutable once
created.

Clients MUST NOT infer store roots from the package. A client MAY be
configured with a set of recognized hash-addressed store roots.
A client that recognizes a store root MUST allow symlinks resolving
into it. A client with no recognized store root MUST apply §4.1(3)
unchanged.
```

Why: Clause 3 protects two things: that a plugin can be moved and
still work, and that an agent client never reads or executes a file the
package author did not ship.

A store reference keeps the second more strongly than containment
does. A store path names its content by digest, so it cannot resolve
to content other than what the builder pinned. A path inside the
plugin root carries no such guarantee on an ordinary mutable
filesystem, where anything with write access can replace it after
install.

It keeps the first in a weaker but nameable form. Copying the plugin
directory alone leaves the link dangling; copying the plugin with its
dependency set does not. The plugin stays relocatable as a closure
rather than as a directory.

The trade is worth stating plainly, because it is the argument:

> A bare `command: "npx"` may be missing, or may silently be a
> different npx. A store reference may be missing — loudly, as a
> dangling path — but can never be a different one.

As written today, clause 3 pushes packages toward the first failure
mode and rejects the second. It rejects exactly the packages whose
provenance is strongest.

Scope: The amendment touches symlink targets only. Every path field
in the format — `command`, `cwd` — stays plugin-relative, and §7.2.1
is untouched.

Cost: An agent client must be configured before it can load such a
package, and the configuration is its operator's decision rather than
the package's. An agent client with no recognized store root behaves
exactly as it does today: under §7.2.2 the affected MCP server entry is
invalid and is dropped, while the plugin's skills keep loading. The
degradation is partial, not fatal.

Not asked for: No exception for file contents. A rewritten shebang
is content, not a path field an agent client resolves, so clause 3
does not reach it and needs no carve-out. See "Shebangs are not a
containment question" below.

Until this is accepted: an agent stack client MAY be configured with
a set of recognized hash-addressed store roots, and if it is, SHOULD
allow a symlink inside a plugin root to resolve into one. That is an
extension to Agent Plugins as published, not conformance to it.

Evidence: The subsections below report what building plugins with the
reference implementation actually produced, and what an agent client
applying Agent Plugins as written does with the result. The example
throughout is a real package, `agent-plugin-agentmemory`, whose
`mcp.json` names `npx`. `$P` stands for the plugin root inside the
store:

```text
/nix/store/<hash>-agent-plugin-agentmemory-0.9.29/
  share/agent-plugins/agentmemory
```

### What a built plugin looks like, and what remains

The upstream package declares a bare command, which resolves to
whatever the consumer happens to have on `PATH`, or to nothing:

```json
{ "mcpServers": { "agentmemory": {
    "type": "stdio", "command": "npx", "args": ["-y", "@agentmemory/mcp"] } } }
```

The builder resolves `npx` at build time and rewrites the command to
a plugin-relative path, as §7.2.1 requires:

```json
{ "mcpServers": { "agentmemory": {
    "type": "stdio", "command": "./bin/npx",
    "args": ["-y", "@agentmemory/mcp"] } } }
```

That resolves against the plugin root to `$P/bin/npx`, inside the
root. Every path field in the package is plugin-relative, every
component is discoverable at its fixed location, and the manifest
validates. What remains outside the plugin root is a single link:

```text
$P/bin/npx -> /nix/store/<hash>-nodejs-22.14.0/bin/npx
```

§4.1(3) permits symlinks only where they resolve within the plugin
root, so that one link is the whole of the non-conformance, and the
whole of what this amendment asks for.

### Shebangs are not a containment question

A skill script's shebang, before and after the build:

```sh
#!/usr/bin/env python3
```

```sh
#!/nix/store/<hash>-agent-plugin-caveman-2.7.0/share/agent-plugins/caveman/bin/python3
```

That string is file content. §4.1 governs paths a client discovers,
reads, or executes from the package; §4.1(5) makes values that are not
defined as paths opaque. A shebang is neither: the client reads the
file as a skill script, and the kernel resolves the interpreter. No
carve-out is needed for this pathway, which is why this amendment does
not ask for one. Files under `assets/` are skipped by the builder and keep
their original shebang.

### Staging links rather than copying

The reference client stages each plugin into a per-launch directory:

```text
<cache>/launch/<agent>/<pid>/agentmemory/
├── .claude-plugin/plugin.json      generated from the spec manifest
├── skills -> $P/skills
└── bin    -> $P/bin
```

The staged root's own children resolve outside it, so the staged tree
fails §4.1(3) exactly as the store tree does.

Copying instead of linking does not fix this. A copied `bin/npx` is
still a link to `/nix/store/<hash>-nodejs-22.14.0/bin/npx`. For the
interpreter to land inside the copied root, the copy would have to
dereference it, which means copying the runtime itself into the
staging directory on every launch. Copy-out relocates the plugin root;
it does not make the package self-contained.

### The clause is enforced on input and broken on output

The builder refuses a source skill that contains an escaping symlink:

```text
symlink skills/foo/data points outside the plugin (-> /etc/passwd);
every path a client reads must resolve inside the skill
```

and then writes `$P/bin/npx -> /nix/store/<hash>-nodejs-22.14.0/bin/npx`
into the output it produces.

That asymmetry is the honest framing of this proposal. The clause is right, and
the implementation agrees with it everywhere except one place, where
it breaks it to gain provenance. The request is not that the clause be
weakened but that it recognize a target whose integrity is established
by digest rather than by location.

## 5. A version for Agent Skills

Target: Agent Skills.

What: Publish the Agent Skills specification with a version
identifier, and tag its releases.

Why: A specification that composes Agent Skills cannot say which
Agent Skills it composes. §4.2 of the Agent Stacks specification
pins a commit hash because there is nothing else to pin, which is
precise but opaque:
a reader cannot tell whether two pins differ materially, and a
conformance claim cannot name what it conforms to. Agent Plugins has
versions and schemas; MCP has dated revisions. Agent Skills is the
one composed specification with neither.

Cost: Release discipline the project does not have today.

Where it belongs: The Agent Skills specification itself, mirroring
the `<major>.<minor>.<patch>` scheme Agent Plugins uses.

## 6. Encode the `command` form in the MCP schema

Target: Agent Plugins.

What: Give `command` in `schemas/<version>/mcp.schema.json` a pattern
that admits only the two forms §7.2.1 defines — a bare executable name
or a path beginning with `./` — as `cwd` already has.

Why: §7.2.1 is prose-only. The schema types `command` as
`{"type": "string", "minLength": 1}` and defers to the specification
text in its description, while `cwd` beside it carries
`^(?:\./|\$\{PLUGIN_ROOT\}(?:/|$)|\$\{PLUGIN_DATA\}(?:/|$))`.
A builder that emitted an absolute `command` passed schema validation
and produced packages whose MCP servers a conforming client would
silently drop. The rule already exists; only its machine-readable form
is missing.

Cost: A package that violated the rule and was previously accepted by
a schema-only validator now fails validation. That is the intent, and
it is a build-time failure rather than a runtime one.

Where it belongs: The MCP schema in Agent Plugins, at the next schema
version.

