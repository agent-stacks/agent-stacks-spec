# Proposal 0003: Secrets

- Status: Draft
- Targets: [spec 0.1.0](../spec/0.1.0.md) §11.3, and the sections it
  would touch: §4, §6.3, §6.4, §7
- Scope: the secrets a stack's plugins read, resolved when the stack
  launches.

## Summary

A stack names the secrets its plugins need and never holds their
values. This proposal composes [SecretSpec](https://secretspec.dev)
for both halves: a plugin declares its secrets in a
`secretspec.toml`, and the consumer keeps the values in whichever
SecretSpec provider they use.

1. **Layout.** A plugin MAY carry `org.agent-stacks/secretspec.toml`,
   naming the secrets it reads.
2. **Declaration.** §6.3 gains `secrets`, which the builder writes to
   that file.
3. **Launch.** Before it runs the agent client, the launcher resolves
   every declared secret from the consumer's provider, asks for any
   that are missing, and refuses to launch without a required one.
   Values reach the agent client's environment and nowhere else.

## Motivation

A plugin's MCP server often needs a credential. The `firecrawl-skills`
plugin's server needs `FIRECRAWL_API_KEY`, and says so in Claude
Code's spelling:

```json
"env": { "FIRECRAWL_API_KEY": "${FIRECRAWL_API_KEY}" }
```

Claude Code fills that in from its own environment. Agent Plugins
does not allow it:

- A client expands only `${PLUGIN_ROOT}` and `${PLUGIN_DATA}` (§9.2),
  so a conformant client hands the server the literal text.
- `env` and `headers` are package data and MUST NOT hold secrets
  (§9.2, §7.2.1).
- There is no way for a plugin to declare a credential. Credential
  storage is left to the client (§7.2.1).

A launcher prototype found a stack's secrets by scanning `mcp.json`
for `${NAME}`, asked for each one the first time the stack launched,
kept it in the macOS Keychain per stack, and exported it at launch.
This proposal keeps the asking: once, at launch, per stack. It
replaces the scan, which reads a form Agent Plugins forbids and
cannot see a variable a server reads without naming it, such as
`FIRECRAWL_API_URL`.

SecretSpec already keeps apart the two things a stack has to keep
apart. A `secretspec.toml` says what a project needs and is safe to
publish. A provider holds the values, and the person running the
project picks it: the system keyring, 1Password, Vault, OpenBao,
Infisical, a cloud secret manager, and others. A stack's author knows
the names. Only the consumer knows where the values are.

## Out of scope

- **Keeping secrets from the agent.** The agent client, its MCP
  servers, and every command it runs can read every value. Keeping a
  value out of the agent's reach needs something outside its process,
  such as a sandbox (proposal 0001) with a proxy that puts the real
  value into outbound requests in place of a placeholder.
- **The agent client's own credentials.** Claude Code's login and
  `ANTHROPIC_API_KEY` are the agent client's to manage.
- **Authorization for remote MCP servers.** Agent Plugins leaves it
  to the client (§7.2.1). See open question 1.
- **Choosing a provider.** A stack does not say where values are kept.

## Proposal

### 1. Composed specification

§4 gains SecretSpec:

- Source: `cachix/secretspec`, release 0.21.1
- File format: `secretspec.toml` at `revision = "1.0"`
- Schema: <https://secretspec.dev/schema/secretspec.schema.json>

It governs the declaration file, the providers, and how one is
resolved against the other.

### 2. Layout

1. A plugin MAY carry `<plugin root>/org.agent-stacks/secretspec.toml`,
   naming each environment variable the plugin's content reads a
   secret from.
2. `org.agent-stacks/` is an Agent Plugins extension directory
   (§8.2). Agent Plugins gives its contents no meaning, so the plugin
   stays conformant.
3. The file SHOULD hold only `[project]`, named for the plugin, and
   `[profiles.default]`. It SHOULD NOT name providers, because a
   plugin cannot know where a consumer keeps values.
4. Each secret SHOULD have a `description`, which the launcher shows
   when it asks for the value. A secret is required unless it says
   `required = false`.

```text
research-stack/
├── bin/
│   └── research-stack
└── share/
    └── agent-plugins/
        └── firecrawl-skills/
            ├── plugin.json
            ├── mcp.json
            ├── skills/...
            └── org.agent-stacks/
                └── secretspec.toml
```

```toml
[project]
name = "firecrawl-skills"
revision = "1.0"

[profiles.default]
FIRECRAWL_API_KEY = { description = "Firecrawl API key" }
FIRECRAWL_API_URL = { description = "URL of a self-hosted Firecrawl", required = false }
```

The declaration lives in the plugin rather than in one file at the
stack root. The secrets belong to the plugin and go with it into
every stack that uses it. Also, a stack assembled from an environment
(§6.5) has no build step that could write a stack-level file, while
each plugin package can bring its own.

### 3. Declaration

§6.3 gains one field:

- `secrets`: the secrets the plugin's content reads, each with a
  name, a description, and whether it is required.

Like `requiredRuntimes`, it records what the plugin reads, not a wish
list. A declaration SHOULD NOT hold a secret's value or say where one
is kept.

§6.4 lists "what the agent may read" among the things a declaration
does not say. This proposal narrows that line: a declaration names
secrets, and never holds a value or names a provider.

### 4. Build

1. A builder SHOULD write `org.agent-stacks/secretspec.toml` from
   `secrets`.
2. An `mcp.json` `env` entry of the form `"NAME": "${NAME}"`, where
   `NAME` is a declared secret, only passes on a variable the server
   already gets from an agent client that passes its environment
   through. A builder SHOULD remove it. That removes the placeholder
   Agent Plugins does not define. The server still depends on a
   variable from its environment, which §9.1 does not allow either;
   Appendix A proposes the amendment.
3. Any other `${NAME}` that names a declared secret, such as one in
   `args`, a renamed `env` entry, or a remote server's `url` or
   `headers`, cannot be handled this way. A builder SHOULD leave it
   as written and report it, naming the file and the field.
4. A builder SHOULD report any other `${NAME}` in `mcp.json` that is
   neither a declared secret nor `${PLUGIN_ROOT}` or `${PLUGIN_DATA}`.
   This is the prototype's scan, moved to build time, where it checks
   the declaration rather than standing in for it.

### 5. Launch

§7 lists four steps. This proposal adds one between resolving the
agent and executing it. When the stack carries a sandbox (proposal
0001), this step wraps the sandbox, so that the sandbox does not need
to reach the consumer's provider.

**Resolve secrets.** When any plugin under `share/agent-plugins/`
carries `org.agent-stacks/secretspec.toml`:

1. **Write a launch spec** into the per-launch staging directory. Its
   project is named for the stack, and it `extends` each plugin's
   `org.agent-stacks/` directory:

   ```toml
   [project]
   name = "research-stack"
   revision = "1.0"
   extends = ["<stack root>/share/agent-plugins/firecrawl-skills/org.agent-stacks"]

   [profiles.default]
   ```

   Because the project is the stack, a provider that keys values by
   project keeps them per stack, as the prototype did. The keyring
   provider stores each value as `secretspec/<project>/<profile>/<name>`.
   A consumer who wants one value shared by every stack sets a
   keyring URI whose prefix leaves out `{project}`.
2. **Check.** Run `secretspec check -f <launch spec>`. In a terminal,
   SecretSpec asks for each missing required secret and stores it
   with the consumer's provider. A launcher with its own interface MAY
   ask there instead, and store each answer with `secretspec set`.
   When it can do neither (`--no-prompt`), it SHOULD refuse to
   launch, naming each missing secret and the plugin that declares
   it.
3. **Run.** Exec `secretspec run -f <launch spec> -- <agent client> [args]`,
   or `-- <sandbox> [options] -- <agent client> [args]` with a
   sandbox.

The provider is the consumer's choice, read from SecretSpec's user
configuration (`secretspec config global init`) or
`SECRETSPEC_PROVIDER`. A launcher MAY default to `keyring://` when the
consumer has configured none.

A launcher that resolves secrets:

1. SHOULD NOT write a value to disk, including the staging directory,
   or pass one as a command-line argument.
2. SHOULD resolve values on every launch, so that a changed secret
   takes effect at the next one.
3. SHOULD give SecretSpec a reason (`--reason "launch <stack>"`).
   SecretSpec refuses access without one when it detects an agent
   (`require_reason = "agents"`, the default), as it does for a stack
   launched from inside an agent session. It records the reason in
   its audit log.

### 6. Bindings

This subsection is non-normative.

In the reference implementation, a plugin's fields are recorded in
its package's `source.json` by the import step. `secrets` goes beside
`requiredRuntimes`:

```json
"secrets": [
  { "name": "FIRECRAWL_API_KEY", "description": "Firecrawl API key" },
  { "name": "FIRECRAWL_API_URL", "description": "URL of a self-hosted Firecrawl", "required": false }
]
```

A stack expression and an environment manifest do not change. They
name the plugin, and the plugin brings its declaration with it.

## Open questions

1. **Tokens in headers.** A remote MCP server that authenticates with
   a fixed header (`Authorization: Bearer ${TOKEN}`) has no conformant
   form, because Agent Plugins forbids expansion in `headers` and
   leaves authorization to the client. Should a launcher pass the
   header through for an agent client that expands it, as Claude Code
   does, with a warning? Or is OAuth the only answer?
2. **Per-plugin delivery.** Every plugin's servers see every secret.
   A launch spec with a SecretSpec scope per plugin
   (`[scopes.<plugin>]`), and each staged server wrapped in
   `secretspec run --scope <plugin>`, would give each server only its
   own secrets and keep values out of the agent client's environment.
   But then each server reads the provider itself, from inside any
   sandbox.
3. **Agent clients that filter the environment.** Claude Code passes
   its whole environment to stdio MCP servers (checked with 2.1.310).
   An agent client that passes only an allow-list needs the launcher
   to name the secrets in that client's configuration when it stages.
4. **Secrets no plugin declares.** These include the token for an
   externally hosted model (proposal 0002, open question 3), or a
   value the consumer adds. A model directory could carry its own
   `secretspec.toml` beside `endpoint.json`, and the launch spec could
   extend that as well.
5. **Launcher or activation.** In the Flox binding, a `[plugins]`
   secrets plugin could resolve secrets at activation instead of at
   launch. Today activation keeps fetched values in environment
   snapshots on disk, which §5 rules out.
6. **Provider policy.** Can an organization require a provider, such
   as its own Vault, for the stacks it publishes?
7. **Namespace.** `org.agent-stacks` assumes the project's domain,
   agent-stacks.org.

## Appendix A: Proposed upstream

Target: Agent Plugins §7.2.1 and §9.1.

What: Let a plugin declare the secrets its content reads, and let a
client supply each one, as an environment variable of that name, to
the plugin's subprocesses. Amend §9.1 so that a plugin may depend on a
base-environment variable it declares as a secret.

Why: §7.2.1 makes credential storage the client's job, but gives a
plugin no way to say it needs a credential. So plugins that need one
use Claude Code's placeholders instead. Until Agent Plugins has a
declaration, `org.agent-stacks/` keeps this one conformant as a client
extension. The server still reads a variable that §9.1 says it must
not depend on.

Cost: A new file or field, and a decision on whether Agent Plugins
names SecretSpec or defines a list of its own.

Where it belongs: The next minor version of Agent Plugins.
