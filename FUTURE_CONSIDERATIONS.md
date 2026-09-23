# Future considerations

This document is non-normative. It records what the
[Agent Stacks specification](spec/0.1.0.md) leaves undecided, and why,
so that a reader can tell a deliberate gap from an oversight. Each
section corresponds to an open item in §7 of the specification.

## Identifying a stack

Nothing at a stack root says that it is a stack. A directory holding
`share/agent-plugins/` and a `bin/` is probably one, and that is as
far as inspection gets. Three things follow:

- A tool cannot recognize a stack without guessing.
- A stack carries no record of the specification version it was built
  against, so an agent stack client cannot tell what it is looking at
  as the specification changes.
- The launcher's name is not well known. A consumer has to know the
  stack's name to find `bin/<stack-name>`.

A small declaration would close all three: the specification version,
the stack's name, the launcher path, and optionally the agent client,
the plugins, and descriptive metadata.

### The constraint any answer has to meet

Stacks are meant to be installed side by side (see
[RATIONALE.md](RATIONALE.md#one-agent-client-per-stack)). With Nix or
Flox, side by side usually means several stacks merged into one
profile, such as `$FLOX_ENV`. The layout already survives that merge
because every path is named per stack or per plugin: `bin/<stack-name>`
and `share/agent-plugins/<plugin-name>/`. A declaration at a fixed
path appears once per merged directory, so the second stack installed
collides with the first. After a merge, the declaration is also the
only record of which plugins belong to which stack.

### Options considered

| Option | Location | Survives merging several stacks |
|---|---|---|
| A. Root file | `<root>/agent-stacks.json` | No: fixed name collides |
| B. Control directory | `<root>/.agent-stacks/`, holding a manifest and a launcher symlink | No: same collision |
| C. File named per stack | `<root>/share/agent-stacks/<stack-name>.json` | Yes: follows the plugin pattern |

A and B are the most visible at the top of a stack. B also leaves
room for more files later, such as hooks or sandbox policy. C is the
only one that keeps the property the rest of the layout has, at the
cost of being one level down rather than at the root.

### Why this is not decided yet

Today the definition of a stack lives in whatever built it — a Nix
expression, a Flox environment, a CLI invocation — and each of those
already knows what it produced. A file restating that is a second
source of truth for as long as nothing reads it, and a format
specified before it has a reader is specified from imagination.

What forces the decision is a stack that travels on its own: installed
by Homebrew or apt, or handed over as a single bundle. At that point
something has to say which launcher to run and which version this is,
and inspection is all the consumer has.

## Stacks composed from stacks

Two cases want the same missing thing. A session manager that runs
other agent clients nests: a stack per agent client, then a stack
whose agent client is the session manager. And a consumer who wants
the same plugins under two agent clients, or one agent client
configured two ways, has two stacks and no vocabulary for saying they
go together.

What an outer stack would have to name, and what a launcher would pass
down to an inner one, is open. Whatever answers this should keep one
agent client per stack: combining stacks is not the same as a stack
that holds two agent clients.

## Sandboxing

A stack bounds what its agent client carries, not what the agent
client can reach once it runs. Whether the sandbox belongs in the
stack, beside it, or outside the specification entirely is open.
Sandbox formats are fragmented — bubblewrap, seccomp, devcontainers,
Docker, vendor sandbox runtimes — so there is nothing to standardize
on yet.

## Secrets

Which secrets an agent client needs to run, and how a stack refers to
them without containing them, is open.

## Further component types

A stack carries one agent client and its plugins today. Other kinds of
content are candidates for later versions or for proposals upstream.

| Component | On disk | Status |
|---|---|---|
| Agent plugin (skills and MCP configuration) | `share/agent-plugins/<name>/` | In 0.1.0 |
| Agent client | a package with a `bin/` | In 0.1.0 |
| Runtimes (interpreters, tools, libraries) | depends on the package manager | In 0.1.0, as dependencies |
| Hooks | scripts and event bindings | Future |
| Stack-level rules | `rules/*.md` | Future |
| Subagents | `agents/*.md` | Future |
| Local models | weights, hash-verified | Future |
| Sandbox configuration | policy the launcher applies | Future, later |
| Permissions and policy | tool allowlists, approval rules | Future, later |
| Tests and evals | `tests/` | Future, later |

**Runtimes** are not a component in their own right: they are
software a plugin needs, carried with it so it travels with the
plugin. A plugin *requires* `python3 >= 3.10`; the package *provides*
`python3 3.12.8`.

**Hooks** are real files, and the reproducibility gain is large: a
hook that runs a linter behaves differently everywhere unless its
dependencies are pinned. Agent client event vocabularies have
converged enough for a neutral manifest mapped at launch. Agent
Plugins defers hooks until formats converge, which makes this a
natural proposal upstream.

**Stack-level rules** are plain text in a format that is already
standard. Most rules files describe the *project*, not the agent
client, and belong in that repository. What belongs in a stack is the
organizational layer, such as house style or security policy.
Precedence between the two stays with the agent client.

**Subagents** are markdown with frontmatter, and agent clients that
support them use roughly the same shape.

**Local models** are the strongest reproducibility case — a stack that
pins its model weights is the same agent everywhere — with real costs:
multi-gigabyte artifacts, per-model licenses, and downloads that must
be hash-verified.

**Sandbox configuration** and **permissions** are security properties
worth their own treatment once a neutral format exists. Until then
they are agent-client-specific configuration.

**Tests and evals**: Agent Plugins already lists plugin validation as
future work. Following its lead is preferable to defining this
separately.
