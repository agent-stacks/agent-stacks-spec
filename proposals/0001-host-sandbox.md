# Proposal 0001: Running a stack's agent client in a host sandbox

- Status: Draft, for discussion. Experimental; like everything before
  1.0.0, anything here may change or be withdrawn.
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.3, and the sections it
  would touch: §6.1, §6.4, §7, §9
- Scope: one kind of sandbox, on one host. Everything else is listed
  under [Non-goals](#non-goals).

## Summary

A stack bounds what its agent client brings with it. It does not bound
what that agent can reach once it runs (§5.3, §11.3). This proposal
adds the smallest thing that changes that for the common case: running
the agent client inside a sandbox on the machine where the stack runs.

Three parts, each small enough to change later:

1. **Layout.** A stack MAY carry one sandbox configuration at
   `<stack root>/share/agent-sandboxes/<sandbox-name>/`, in that
   sandbox's own format.
2. **Declaration.** The vocabulary in §6.1 gains an optional `sandbox`
   field. A builder satisfies it by placing the configuration above, in
   whatever form the builder uses for everything else.
3. **Launch.** When a stack carries a sandbox, the launcher writes the
   configuration the sandbox will actually enforce, which is the stack's
   base plus what the stack implies plus the consumer's overrides, into
   its per-launch staging directory, and then executes
   `<sandbox> [options] -- <agent client> [args]` instead of the agent
   client alone.

No new policy format is proposed. Each sandbox keeps its own.

## Motivation

### The agent client is the thing to sandbox

An agent client starts its plugins' MCP servers and hooks as its own
child processes, and runs its tools (read, edit, shell) in-process or
as children. A sandbox applied to the agent client's process therefore
covers all of them. A sandbox applied to anything narrower does not.

The built-in sandboxes some agent clients ship show why this matters.
Claude Code's `/sandbox` fences the shell commands it runs; its MCP
servers and hooks run outside it. A stack's plugins are mostly MCP servers,
hooks, and skills that run scripts. A sandbox around the whole process
is the one that reaches them.

### Why on the host first

Sandboxes come in many shapes: OS process profiles, containers,
microVMs, full VMs, hosted APIs, egress proxies, and vendor-hosted
agents that run their own harness. Most of them cannot see the
consumer's machine, which means they need the stack delivered in a
different form, usually a Linux build or an image. That is real work,
and none of it is needed to answer the first question a consumer asks:
can I run this stack so that the agent cannot read my SSH keys or call
arbitrary hosts?

A host sandbox answers that with the stack exactly as it is. It sees
the same files, so the stack's plugins and runtimes in the
content-addressed store are already where the agent expects them. Two
sandboxes of this kind run on both macOS and Linux and accept the
handoff this proposal needs:

- [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime)
  (`srt`): Seatbelt on macOS, bubblewrap and seccomp on Linux, with a
  domain-filtering proxy.
- [nono](https://nono.sh): Seatbelt on macOS, Landlock on Linux, with
  a domain-filtering proxy.

Appendix A reports what they did when tested against a real stack.

## Non-goals

- Containers, microVMs, full VMs, remote and hosted sandboxes. They
  need the stack delivered as an image or a Linux build, which §11.1's
  open question about a self-describing stack bears on.
- Egress gates as a separate layer. Both sandboxes above filter by
  hostname already.
- A common, sandbox-neutral policy format. See
  [Considered alternatives](#considered-alternatives).
- Combining two sandboxes by nesting one inside the other. On macOS
  only one Seatbelt profile applies to a process, so it fails there
  (Appendix A).
- Agents that run inside a vendor's own runtime, where the stack never
  launches the agent client.

## Proposal

### 1. Layout

1. A stack MAY carry a sandbox configuration at
   `<stack root>/share/agent-sandboxes/<sandbox-name>/`.
2. A stack SHOULD carry at most one. A stack names one agent client
   (§5.4), and one sandbox around it; two would ask the launcher to
   choose, which is the ambiguity §5.4 rules out.
3. The directory SHOULD hold that sandbox's configuration in the
   sandbox's own format, under the file name the sandbox documents or,
   where it documents none, the name listed for it in Appendix B.
4. The configuration is a base, not the final policy. It SHOULD hold
   what the stack's author decided: which hosts beyond the stack's own
   needs to allow, which paths to deny. What the stack itself implies
   is added at launch (§3 below), so the author does not restate it.

```text
caveman-stack/
├── bin/
│   └── caveman-stack
└── share/
    ├── agent-plugins/
    │   └── caveman/...
    └── agent-sandboxes/
        └── nono/
            └── profile.json      nono's own profile format
```

This is the same move §5.1 makes for plugins: a directory under
`share/` per thing, discoverable by listing it, each describing
itself in its own format. A stack with no `agent-sandboxes/` directory
runs exactly as it does today.

### 2. Declaration

§6.1 gains one optional field:

- `sandbox` — the sandbox to run the agent client under, and its base
  configuration.

The carried and borrowed distinction of §6.2 applies to the sandbox's
executable as it does to the agent client's. A stack that carries
`nono` in its dependency set runs the same sandbox everywhere; a stack
that relies on the host's runs whatever is installed there, and the
launcher SHOULD say so, as §9.3 asks for the agent client.

§6.4 currently lists "a sandbox" among the things a declaration does
not say. This proposal replaces that line with the narrower statement
that a declaration does not say what any sandbox other than this one
allows, and does not choose between sandboxes at launch.

### 3. Launch

§7 lists four steps. This proposal adds one between resolving the
agent and executing it.

**Wrap.** When `share/agent-sandboxes/` holds a sandbox:

1. **Resolve the effective configuration** from three inputs, in
   this order:
   - the stack's base configuration;
   - what the stack implies, which the launcher already knows from the
     scan in step 1: read access to the dependency set, the hosts named
     in the plugins' `mcp.json`, and what the agent client itself needs
     to run (its API and authentication hosts, its state directories);
   - the consumer's overrides (§4 below).
2. **Write it** into the per-launch staging directory §7 already
   creates, never inside the stack.
3. **Execute** `<sandbox> [options] -- <agent client> [args]`, passing
   the staged plugins to the agent client exactly as it would without
   a sandbox.

A launcher that wraps an agent client:

- SHOULD refuse to launch, with a message naming the reason, when the
  sandbox cannot run on this system or cannot honor part of the
  effective configuration. It SHOULD NOT fall back to running the agent
  unsandboxed without being told to.
- SHOULD make the effective configuration inspectable before launch,
  for example with a command that prints the file it would write.
- SHOULD prepare what the sandbox cannot grant later. Landlock, for
  one, cannot grant access to a path that does not exist when the
  sandbox starts, so an agent client's state directory has to be
  created first.

The per-sandbox work, meaning what "read access to the dependency set"
or "an allowed host" looks like in this sandbox's format, belongs in a
small adapter per sandbox. Whether that adapter lives in the agent
stack client or ships beside the configuration is left to
implementations; Appendix B lists what each of the two tested sandboxes
needs from it.

### 4. Consumer overrides

A consumer will want to adjust a stack's sandbox without rebuilding
the stack: allow one more host, read one more directory, or turn the
sandbox off to debug. The launcher MAY accept, in increasing
precedence:

1. a consumer overlay file in the sandbox's own format;
2. command-line flags or environment variables, including one that
   disables the sandbox for a launch, which the launcher SHOULD
   announce on every launch where it applies.

The two sandboxes compose differently, which is why the merge belongs
to the launcher rather than to the sandbox: nono profiles inherit
through `extends`, but only by name and not by path, while
sandbox-runtime reads exactly one settings file and does not layer
them. Writing one resolved file per launch works the same for both.

Whether a stack author can set limits a consumer cannot loosen is left
open (see [Open questions](#open-questions)).

### 5. Bindings

This subsection is non-normative. It follows §6.5: the same field,
spelled the way each builder spells everything else.

As a Nix function argument:

```nix
mkAgentStack {
  name = "caveman-stack";
  harness = claude-code;
  plugins = [ agent-plugin-caveman ];
  sandbox = agent-sandbox-nono;
}
```

As an environment manifest, where the environment is the declaration
and installing a package that provides `share/agent-sandboxes/nono/`
is how the field is set:

```toml
[install]
claude-code.pkg-path          = "claude-code"
agent-plugin-caveman.pkg-path = "agent-stacks/agent-plugin-caveman"
agent-sandbox-nono.pkg-path   = "agent-stacks/agent-sandbox-nono"
```

As a command line, for a stack assembled for the duration of one
launch:

```sh
agent-stacks launch --sandbox nono claude
```

The first two put the choice on disk, where a stack's author can
distribute it and a consumer can inspect it before running anything.
The third is the override and the debugging path.

## Considered alternatives

**A command-line flag only.** The quickest to build, and kept as the
override. On its own it fails the people the stack is for: an author
cannot ship a sandboxed stack, a consumer cannot tell before launch
whether one will be applied, and a consumer who starts the agent client
directly rather than through a launch command never passes the flag.

**A field in a stack manifest.** A `stack.json` naming the sandbox
would be one obvious place. §11.1 explains why there is no stack
manifest yet, and this proposal does not need one: listing
`share/agent-sandboxes/` answers the same question. If a manifest
arrives, it can name the directory that is already there.

**A common policy format.** One sandbox-neutral description of what an
agent needs (files, hosts, credentials) that every sandbox adapter
translates. It is the right shape once several sandboxes of different
kinds are supported, and it is deferred until then, because a format
specified before two sandboxes have been driven from it would be
specified from imagination, the same reasoning §11.1 gives for the
stack manifest. Each sandbox's own format is inspectable today, and is
documented by the people who maintain the sandbox.

**The agent client's own sandbox.** Rejected for stacks, for the reason
in the Motivation: it covers less of what a stack carries, and it is
specific to one agent client, while a stack's sandbox should not change
when its agent client does.

**Nesting two sandboxes** (one for files, one for network). Rejected:
on macOS a process gets one Seatbelt profile, and every nesting we
tried failed. Both tested sandboxes filter hosts themselves, so there
is nothing left for a second one to do on the host.

## Rollout

Each step is usable on its own, and each tells the next what to keep.

1. **One sandbox, end to end.** nono, on macOS and Linux: a stack
   carries `share/agent-sandboxes/nono/`, and starting the agent client
   runs it inside nono with the stack's plugins. nono first because it
   is the one tested sandbox that needs no administrator access on
   either OS (Appendix A).
2. **A second sandbox.** sandbox-runtime. It needs no administrator
   access on macOS, and on Ubuntu 24.04 and later one host change that
   an administrator or fleet tooling makes once (Appendix A). A second
   sandbox with a different configuration model is the test of whether
   the adapter boundary in §3 is in the right place.
3. **Overrides and inspection.** The overlay file, flags, the disable
   switch, and printing the effective configuration.
4. **Into the specification.** Fold what survives into a numbered
   version, with the open questions below answered or restated.

## Open questions

1. **Who wins, author or consumer?** Can a stack's author set limits a
   consumer cannot loosen, such as hosts that stay blocked? A platform
   team distributing a stack will want that; a developer debugging one
   will not. nono can partly express it; sandbox-runtime cannot without
   the launcher enforcing it.
2. **Where does the adapter live?** In the agent stack client, keyed by
   sandbox name, or in a package beside the configuration that the
   launcher calls through `<adapter> [options] -- <command>`?
3. **What does an agent client need?** Its API and authentication hosts,
   state directories, and terminal access are facts about the agent
   client, not the stack, and change with its releases. Where should
   they be recorded so every adapter does not rediscover them?
4. **Two sandboxes in one stack.** This proposal says SHOULD NOT. Is
   refusing to launch the right response when it happens anyway, or
   should an override pick one?

## Appendix A: What the two sandboxes did

This appendix is non-normative. sandbox-runtime 0.0.79 and nono 0.78.0,
both installed as packages, ran Claude Code 2.1.289 with the `caveman`
plugin from the store, on macOS (aarch64) and Ubuntu 26.04 (aarch64,
kernel 7.0), 8 October 2026. Every check below was run on both, in a
script, headless and in an interactive terminal session.

### A.1 Platform

| | nono, macOS | srt, macOS | nono, Linux | srt, Linux |
|---|---|---|---|---|
| Mechanism | Seatbelt | Seatbelt | Landlock | bubblewrap + seccomp |
| Runs without administrator access | yes | yes | yes | **no, on Ubuntu 24.04+** |
| Agent client + store plugin, headless | yes | yes | yes | yes, after the host change |
| Interactive session (keys, resize, Ctrl-C, exit status) | yes | yes, with `allowPty` | yes | yes, after the host change |

Ubuntu 24.04 and later set `kernel.apparmor_restrict_unprivileged_userns=1`,
which sandbox-runtime's README documents. bubblewrap from the store then
fails with `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`.
Either of two root-level changes fixes it, and both were tested:

- an AppArmor profile granting `userns` to the store path of the
  sandbox's `bwrap` (its seccomp helper inherits the profile), which
  has to be regenerated whenever that store path changes; or
- setting the sysctl to `0`, which relaxes the restriction for the
  whole host.

### A.2 Configuration

| | nono | sandbox-runtime |
|---|---|---|
| Format | JSON profile with a schema and a validator | flat JSON, strict schema (empty lists required) |
| Loaded from a read-only store path | yes, `--profile <path>` | yes, `--settings <path>` |
| Invalid or missing configuration | refuses to run | refuses to run |
| Inheritance | `extends`, by name only; a store path is rejected | none; a second `--settings` replaces the first |
| Consumer's own file | `~/.config/nono/profiles/` | `~/.srt-settings.json`, ignored when `--settings` is given |
| Command-line additions | `--allow`, `--read`, `--allow-domain`, … | none |
| Narrowing an inherited grant | macOS yes; Linux refuses to start | n/a |

### A.3 Defaults

| | nono | sandbox-runtime |
|---|---|---|
| Network with no rules | **open** | closed |
| Reads outside the project | denied unless granted | **allowed** unless denied |
| `~/.ssh` | denied by a built-in group | denied only if listed |
| Writes outside the project | denied | denied |
| Hostname allowlist | enforced through its proxy | enforced through its proxy |

The two disagree on exactly the defaults a stack author would assume,
which is an argument for the launcher writing the effective
configuration out in full (§3) rather than relying on either default.

### A.4 What an adapter had to add

Each of these was found by a failure, not by reading documentation:

- Read access to the content-addressed store. On Linux, nono's default
  profile does not grant it, and the stack's binaries cannot start.
- Claude Code needs `platform.claude.com` as well as
  `api.anthropic.com`: the first is where it refreshes its login. With
  only the API host, sessions fail with `401` once the token expires.
- Claude Code's state directories, created before the sandbox starts
  on Linux, where Landlock cannot grant a path that does not exist yet.
- sandbox-runtime on macOS needs `"allowPty": true` for an interactive
  session. Without it the terminal cannot be put into raw mode: input
  echoes escape sequences, keys are lost, and one Ctrl-C ends the
  session.
- `--` between the sandbox's options and the command for
  sandbox-runtime, which otherwise reads the agent client's flags as
  its own.
- nono's `--no-diagnostics`, or the session ends in an interactive
  review prompt after the agent exits.

## Appendix B: Configuration files by sandbox

This appendix is non-normative, and grows with each sandbox a stack
can carry.

| Sandbox | Directory | File | Passed as |
|---|---|---|---|
| nono | `share/agent-sandboxes/nono/` | `profile.json` | `nono run --profile <file> -- …` |
| sandbox-runtime | `share/agent-sandboxes/srt/` | `settings.json` | `srt --settings <file> -- …` |
