# Agent Stacks Specification

A stack is an agent client, its plugins, and the runtimes both need,
built and delivered as one unit.

This repository holds the specification. It is a meta specification:
it composes the [Agent Plugins](https://github.com/agentplugins/agent-plugins-spec),
[Agent Skills](https://agentskills.io/specification), and
[Model Context Protocol](https://modelcontextprotocol.io/specification)
specifications, pins the versions it composes, and describes what a
stack adds on top of them.

## Versions

- [0.1.0](spec/0.1.0.md) — draft, current

## Companion documents

These are non-normative:

- [RATIONALE.md](RATIONALE.md) — why the specification makes the
  choices it does
- [FUTURE_CONSIDERATIONS.md](FUTURE_CONSIDERATIONS.md) — what is not
  decided yet
- [UPSTREAM_PROPOSALS.md](UPSTREAM_PROPOSALS.md) — changes proposed to
  the composed specifications

## Licensing

See [LICENSE.md](LICENSE.md).
