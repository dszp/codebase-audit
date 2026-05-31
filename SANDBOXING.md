# Runbook — Auditing an Untrusted Repo Safely

This skill reads the target repo's own files (`README`, `CLAUDE.md`,
`AGENTS.md`, `.cursor/rules/*`, source, dependency manifests) as part of its
comprehension pass. If you point `/audit` at a repo you don't trust, any of
those files can carry **prompt injection** — text crafted to hijack the
auditing agent into exfiltrating data, writing outside the repo, or running
commands. The skill's own content is clean; this risk is inherited from the
*audited code*, and it applies to any agent run against untrusted input.

This runbook contains it.

## The one thing that matters: isolate the whole process, not just bash

Claude Code can run shell commands in an OS sandbox, but its first-class tools
— `Read`, `Write`, `WebFetch` — run **in the agent process, not through bash**.
A bash-only sandbox stops a malicious `curl`; it does nothing about the agent
being talked into `WebFetch`-ing your data to an attacker host or `Write`-ing
outside the repo. So the boundary has to wrap the entire session: a container
or VM, not a shell wrapper.

## Procedure

1. **Clone the untrusted repo to a throwaway location** — never your normal
   workspace, and never somewhere a parent directory holds other projects:

   ```
   mkdir -p /tmp/untrusted && git clone --depth 1 <repo-url> /tmp/untrusted/target
   ```

2. **Open the session inside a devcontainer.** Run the `devcontainer-setup`
   skill to scaffold one (Anthropic ships a reference container with a firewall
   init script). Mount **only** the cloned repo. Do not mount the host home
   directory.

3. **Withhold host credentials.** Do not pass `~/.ssh`, `~/.aws`, cloud tokens,
   or a broad `gh` token into the container. An injection can't steal a secret
   that isn't present.

4. **Disable every MCP server.** Any MCP servers connected to your normal
   Claude session are pre-authenticated to whatever data and accounts they
   front — and frequently expose **write** tools (sending messages, creating
   or deleting records, executing remote API calls). An injected audit could
   read that data or act on those accounts through them, and the egress
   allowlist does **not** stop this: remote connectors ride your
   already-authenticated Claude session rather than a firewalled domain, and
   local (stdio) servers run in-process. This is a separate boundary from the
   network firewall and must be closed explicitly.

   Launch the session with all MCP config ignored:

   ```
   claude --strict-mcp-config        # loads ONLY servers from --mcp-config;
                                      # with no --mcp-config, that means none
   ```

   In a container, also simply **do not copy `~/.claude.json` or any project
   `.mcp.json`** into it — absent config, no servers spawn. Either way, confirm
   with `/mcp` that the server list is empty before reading any repo file.

5. **Restrict network egress to an allowlist.** This is the load-bearing
   control — exfiltration needs network, and the skill's Live Discovery
   legitimately wants *some* network for doc lookups. Allow only the hosts the
   skill actually fetches; deny everything else. Current allowlist (the
   authoritative sources referenced across the skill's modules):

   ```
   # Standards / docs
   www.w3.org  developer.mozilla.org  web.dev  caniuse.com  browsersl.ist
   web-platform-dx.github.io  schema.org  www.sitemaps.org  datatracker.ietf.org
   # Security / CVE
   owasp.org  cheatsheetseries.owasp.org  genai.owasp.org  nvd.nist.gov
   # Runtime / EOL / registries (for `npm view`, `pip`, etc.)
   endoflife.date  nodejs.org  devguide.python.org  pypi.org  crates.io
   kotlinlang.org
   # Vendor docs
   docs.claude.com  platform.openai.com  ai.google.dev  developers.cloudflare.com
   developers.google.com  developer.apple.com  developer.android.com
   docs.stripe.com  docs.sentry.io  vercel.com  neon.com  workos.com  www.workos.com
   support.google.com
   # GitHub (clone + `gh api`, only if you choose to allow it)
   github.com  api.github.com
   # Compliance / government refs (add per the frameworks you actually run)
   www.hhs.gov  www.pcisecuritystandards.org  www.iso.org  www.fedramp.gov
   www.ftc.gov  www2.ed.gov  studentprivacy.ed.gov  www.section508.gov
   commission.europa.eu  digital-strategy.ec.europa.eu  ec.europa.eu
   www.edpb.europa.eu  artificialintelligenceact.eu  www.esma.europa.eu
   www.eiopa.europa.eu  www.priv.gc.ca  www.cai.gouv.qc.ca  crtc.gc.ca
   fightspam.gc.ca  www.ipc.on.ca  oipc.ab.ca  www.ombudsman.mb.ca
   www.ontario.ca  oag.ca.gov  www.aicpa-cima.com  www.gov.br  www.meity.gov.in
   cldr.unicode.org  www.swift.org
   ```

   If you'd rather not maintain a firewall, the safe degenerate case is **deny
   all egress** and accept that every Live Discovery step falls back to
   embedded knowledge (the skill already handles this — it tags findings
   `may be stale, last-known: <date>` rather than fabricating currency).

6. **File tickets from the host, not the sandbox.** Module 00.3 wants `gh` to
   create issues, but you don't want a GitHub token inside a session running
   untrusted code. Run the sandboxed audit with `--no-tickets` (or
   `--dry-run`), review the drafted ticket list, then file from the host after
   the injection risk has passed:

   ```
   /audit --no-tickets
   ```

7. **Don't rely on permission prompts as the backstop.** If your settings have
   `skipAutoPermissionPrompt: true` (or you run in an auto-approve mode), the
   agent won't pause before acting — so the container boundary is doing *all*
   the work. Either confirm the container is real isolation, or drop that
   setting for untrusted-repo sessions.

## Quick reference

| Control | Stops | Skip it and you lose |
|---|---|---|
| Container / VM around the session | Writes + fetches escaping the repo | Everything below becomes the only defense |
| No host credentials mounted | Secret theft | Injection can read & transmit your keys |
| All MCP servers disabled (`--strict-mcp-config`) | Reads/writes to your connected data & accounts | Connected services become injection tools |
| Egress allowlist (or deny-all) | Data exfiltration | A fetch tool becomes an exfil channel |
| `--no-tickets` in sandbox | Token abuse via `gh` | Scoped/broad token exposed to untrusted code |

## When you can skip all this

This is for **untrusted** code. Auditing your own repos, a client's repo under
an engagement, or anything you'd run `npm install` on without a second thought
needs none of the above — run `/audit` directly. The threat is specifically
attacker-authored content in a repo you have no basis to trust.
