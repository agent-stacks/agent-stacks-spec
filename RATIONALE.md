# Rationale

This document is non-normative. It explains choices made in the
[Agent Stacks specification](spec/0.1.0.md), so that the specification
itself can stay short.

## Why a meta specification rather than a new format

A stack is a composition of three existing specifications. Almost
every requirement a builder or an agent client has to meet is already
written down in one of them. A fourth set of requirements would be a
specification for a specification: more text to keep in sync, and a
second authority for readers to reconcile. What was missing was not
rules but the composition — which specifications, which versions, and
how they fit.

## One agent client per stack

A stack names one agent client. Not a list, not a default with
alternatives, not a set to choose from at launch.

Read positively, that is the whole shape of a stack: pick the agent,
then configure it. The plugins it loads, the runtimes it needs, and —
as the specification grows — the sandbox it runs under and the
secrets it may reach are all configuration *around* that one agent.
A stack with two agents in it has no answer to the only question a
consumer asks, which is what happens when they run it.

Three reasons the rule is worth stating rather than assuming:

1. A stack is an instance, not a menu. "Run this" is unambiguous when
   there is one thing to run.
2. The alternative multiplies. Every configuration axis added to a
   stack that holds several agents multiplies against them: three
   agents and three sandboxes is nine combinations to describe, most
   of which no one wants and none of which anyone declared.
3. Wanting several is already expressible. Build one stack per agent
   and install them side by side; the plugins they share are the same
   packages either way.

An agent client that runs other agent clients — a session manager
such as Agent Deck — is composed the same way, by nesting rather than
by widening: a stack per agent, then a stack whose agent client is the
session manager. How the outer stack names the inner ones is an open
item; see [FUTURE_CONSIDERATIONS.md](FUTURE_CONSIDERATIONS.md).

## What a stack is not

- **Not a plugin.** A plugin is a package of components, specified by
  Agent Plugins. A stack is a plugin's delivery vehicle, together with
  an agent client to run it and the runtimes both need.
- **Not a per-agent-client configuration tree.** A stack holds the
  neutral layout. `.claude/`, `.codex/`, and their equivalents are
  produced at launch and belong to the launch, not to the stack.
- **Not an environment or a package manager.** A stack does not manage
  the consumer's `PATH`, shell, or project. It carries what its agent
  client and plugins need and leaves the rest of the machine alone.
- **Not a sandbox.** A stack bounds what its agent client brings with
  it, not what the agent client can reach once running.

## Why the layout is adapted at launch

A stack's layout is deliberately not what any agent client reads. The
launcher turns it into whatever the agent client in the stack expects,
at the moment it runs. That indirection is why a stack does not have
to be rebuilt when an agent client changes its configuration format,
and why one plugin tree serves agent clients that disagree about
everything else.

Because the staged form is derived and disposable, the launcher never
writes inside the stack. The same stack can serve two agent clients at
once, and two launches of the same agent client do not share staging.

## Why the launch is mostly recommendations

How plugins and an agent client combine into something runnable is
largely a client decision. The reference implementation makes one set
of choices — neutral layout in the package, adaptation at launch — and
those choices are worth writing down because another implementer will
face the same questions. They are written as SHOULD rather than MUST
so that a client with different constraints is not forced to violate
the specification to do its job.

Warning when the agent client came from the consumer's environment is
a warning and not a failure: running the host's agent client is a
legitimate thing to want, but an unreproducible stack should announce
itself rather than look identical to a reproducible one.

## Why no MCP revision is composed directly

`mcp.json` names transports, and client and server negotiate the
protocol version when they connect, so the revision a stack ends up
speaking is a property of the server it launches rather than of the
stack. The specification names the current revision for reference and
`2024-11-05` only because that revision defines the legacy HTTP+SSE
transport that Agent Plugins' `sse` transport type selects.

## Why upstream proposals are kept separate

Several things the reference implementation does would change
specifications this project does not own. Writing them as requirements
here would manufacture a conflict: a package following this
specification would fail Agent Plugins as published. Keeping them in
[UPSTREAM_PROPOSALS.md](UPSTREAM_PROPOSALS.md) states what we want and
why, while leaving the published specifications the authority until
they say otherwise.
