# Loopback GitHub Action

Runs Loopback jobs (triage, analysis, fix) on your own GitHub Actions runners, so your code and
your AI provider key never leave your infrastructure. It also uploads your build's source maps,
so that minified stack traces in tickets point to real files and lines.

Code understanding and verification are powered by [vexp](https://vexp.dev).

## Running jobs

Loopback starts a job by sending a `repository_dispatch` event (`loopback-job`) whose
`client_payload` carries only the job id and the API URL. The onboarding pull request adds this
workflow for you:

```yaml
# .github/workflows/loopback.yml
name: Loopback
on:
  repository_dispatch:
    types: [loopback-job]
permissions:
  contents: read
  id-token: write          # signs in to Loopback with GitHub OIDC
concurrency:
  group: loopback-${{ github.event.client_payload.job_id }}
jobs:
  run:
    runs-on: ubuntu-latest   # or [self-hosted]
    timeout-minutes: 60      # above the job time limit of the workspace (at most 120 minutes)
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
          persist-credentials: false
      - uses: Loopback-so/action@v1
        with:
          agent-key: ${{ secrets.LOOPBACK_AGENT_KEY }}
```

What one run does:

1. Reads the job id from the dispatch event and requests a GitHub OIDC ID token with audience
   `loopback`.
2. Exchanges it at `POST /v1/runner/oidc` for two tokens. The job token is scoped to that one job
   of that one repository and expires after 60 minutes. The agent token lets the agent use only
   the Loopback tools of that job. Loopback checks the ID token's signature, issuer, audience and
   `repository_id`, and that the job belongs to the repository. Each ID token is accepted once:
   every exchange uses a new one.
3. Downloads the job (ticket, logs, diagnosis) with the job token, then prepares the checkout in
   `GITHUB_WORKSPACE` at the job's base commit.
4. Runs the agent configured for the workspace or in `.loopback.yml`. It streams progress and
   heartbeats to Loopback, so you can follow the job, or cancel it, from the dashboard. When fewer
   than 10 minutes of the job token are left, the harness renews it with Loopback
   (`POST /v1/jobs/:id/token`); if Loopback no longer renews it, the action signs in again with a
   new ID token for the same job. A job runs for at most 120 minutes.
5. For a fix, the harness verifies the change (vexp `verify_done`) and runs your tests. It commits
   only when both pass. It then pushes the `loopback/…` branch, and Loopback opens the pull request
   unless the workspace autonomy is `copilot`.
6. Reports the result. Reporting is idempotent: re-running the workflow never creates a second
   pull request.

When Loopback chains jobs (a triage followed by its fix), the next job runs in the same workflow
run, with its own OIDC exchange.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `agent-key` | | API key of your AI provider (Anthropic `sk-ant-…`, OpenAI, OpenRouter `sk-or-…`), as a secret. Only the agent process receives it. Leave it empty for Bedrock or Vertex (see below) or for a Codex ChatGPT login stored in Loopback. |
| `api-url` | `https://api.loopback.so` | Loopback API. The OIDC token is sent only here: a dispatch that names another API is refused. |
| `oidc-audience` | `loopback` | Audience of the GitHub OIDC token. Change it only when your Loopback install verifies another one (`GITHUB_OIDC_AUDIENCE`); the onboarding pull request then writes it for you. |
| `command` | `run` | `run` executes the dispatched job. `upload-sourcemaps` uploads source maps. |
| `sourcemaps-path` | `dist` | `upload-sourcemaps`: build directory, relative to the repository, that holds the `.map` files. |
| `url-prefix` | `/` | `upload-sourcemaps`: URL path the build directory is served under (for example `/static`). |
| `release` | | `upload-sourcemaps`: the release your app reports to Loopback (for example `my-app@1.8.3`). Required for that command. |
| `github-token` | | Optional. A token to fetch history that a shallow checkout lacks. Without it, the harness asks Loopback for a short-lived read-only token. It is never given to the agent and never used to push. |

## Outputs

| Output | Description |
| --- | --- |
| `status` | Outcome of the last job: `succeeded`, `needs_human`, `failed` or `cancelled`. |
| `job-id` | Id of the last job run. |
| `branch` | Fix branch pushed, if any. |
| `pr-url` | Pull request of the fix, if one was opened. |
| `uploaded` | `upload-sourcemaps`: number of source maps uploaded. |

`needs_human` means that the ticket went back to your team, for example because the fix did not
pass your tests. The workflow run stays green. A step failure means that the job could not run or
that its result could not be reported.

## Agents and credentials

The agent is chosen in Loopback's workspace settings or in `.loopback.yml` (`agent.name`,
`agent.auth`). The allowed combinations:

- **builtin** (default): Loopback's own agent, using the provider of your key (Anthropic, OpenAI,
  OpenRouter), Bedrock or Vertex.
- **claude-code**: an Anthropic API key, Bedrock or Vertex. Claude subscription logins (claude.ai
  OAuth tokens) are refused outside MCP mode, as Anthropic's terms require. Claude Code runs in
  `--bare` mode, which has no tool to create files: the harness gives it a `write_file` tool,
  confined to the repository like the builtin agent's. Install `bubblewrap` before the Loopback
  step (see below).
- **codex**: an OpenAI API key, or a ChatGPT login that you store in Loopback. The harness gives
  the login to the agent for the run and sends the refreshed login back afterwards. Codex runs
  only inside `bubblewrap` (see below); its commands get Codex's own sandbox, read-only outside a
  fix. `LOOPBACK_CODEX_SANDBOX=danger-full-access` is refused.
- **opencode**: the API key of its provider.

The action installs a missing agent CLI and the vexp engine with npm, at pinned versions, into
the job's temporary directory. On self-hosted runners, you can preinstall the same versions to save
time, or use the `loopback-runner` container instead of this action.

### bubblewrap: the sandbox of your tests

Your tests are code the agent can change, and they run as the runner's user. The action's own
process keeps the agent key and the OIDC request token in its starting environment
(`/proc/<pid>/environ`), and on GitHub-hosted runners that user has passwordless sudo and the
Docker socket. So the harness runs your dependency installs, tests and TypeScript type-check only
inside `bubblewrap`: in their own PID namespace (no other process is visible, and nothing they
start outlives them), with setuid programs disabled, with the Docker socket, `/tmp`, the job's
temporary directory and the cloud credential files of earlier steps out of reach. Without a
working bubblewrap they do not run at all: the job log says why, and the fix is reported with its
tests "not run" and the reason.

On a GitHub-hosted runner, a disposable VM, the action installs `bubblewrap` when it is missing and
allows unprivileged user namespaces (`kernel.apparmor_restrict_unprivileged_userns=0` on Ubuntu
24.04) before the job. A self-hosted runner is never changed: install `bubblewrap` and allow user
namespaces there yourself. bubblewrap runs on Linux only: on macOS and Windows runners the tests
are reported as not run.

Claude Code also runs the commands of the agent (your test command, read-only git commands)
through `bubblewrap`, which removes the credentials from their environment. Without it, the agent
does not run your tests itself, and the job log shows a warning. Codex runs only inside
`bubblewrap`, in its own PID namespace, with the SSH keys, cloud and registry logins and a
self-hosted runner's registration (`.credentials`) hidden; without it a Codex job does not start
and asks for a person. opencode's shell has no sandbox:
it never runs your test command, the harness does. On a self-hosted runner, install it in a step
before the Loopback step:

```yaml
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
          persist-credentials: false
      - run: sudo apt-get install -y bubblewrap
      - uses: Loopback-so/action@v1
        with:
          agent-key: ${{ secrets.LOOPBACK_AGENT_KEY }}
```

The harness checks bubblewrap at every run. It also needs unprivileged user namespaces. If the job
log says that bubblewrap cannot create namespaces, the runner blocks them. On Ubuntu 24.04 this is
AppArmor's `kernel.apparmor_restrict_unprivileged_userns`. On a self-hosted runner, give
`/usr/bin/bwrap` an AppArmor profile that allows user namespaces. Test suites that need Docker
(for example Testcontainers) cannot reach it from the sandbox: start their services in an
earlier step instead.

### Bedrock and Vertex

For **Bedrock**, add `aws-actions/configure-aws-credentials` (with OIDC) before this step and set
`agent.auth: bedrock`. For **Vertex**, add `google-github-actions/auth` and set
`ANTHROPIC_VERTEX_PROJECT_ID` with `agent.auth: vertex`. Only the provider's own variables are
passed to the agent.

## Security model

- **No long-lived Loopback secret in your repository.** The action signs in with GitHub OIDC. The
  job token it receives is scoped to one job and expires in 60 minutes. Renewals never outlive the
  job's time limit and never add permissions. Only fix jobs can get a push token: the tokens of
  triage jobs and of external pull request analyses cannot.
- **The job token stays with the harness.** The agent gets its own token for Loopback's MCP
  server, valid only for the tools of its job. It cannot fetch git tokens, post results or reach
  other jobs.
- **Every token is masked** in the workflow log as soon as the action holds it: the ID tokens, the
  job token and its renewals, the agent token, the agent key and the git tokens of the job.
- **The agent cannot push.** Before the agent starts, the harness removes every credential
  persisted in the checkout: `actions/checkout` extra headers (in `.git/config` and the
  `includeIf` credential files in `RUNNER_TEMP`), credential helpers, `insteadOf` URLs, tokens in
  remote URLs, SSH keys and submodule configs. `persist-credentials: false` is still recommended.
  The harness alone pushes, with a short-lived token from Loopback, and only to the job's
  `loopback/…` branch (`HEAD:refs/heads/loopback/…`). It never pushes to your default branch, and
  it never force-pushes outside `loopback/`.
- **The agent runs isolated.** It gets a private home and config directory, no global git config,
  and an environment without the action's inputs, `GITHUB_TOKEN` or the OIDC request token. Its
  git commands run without repository hooks and without any transport, so a `git push` of the
  agent fails. Claude Code and opencode may run read-only git commands only: no `git add`, no
  `git commit`, and `git diff` only on the repository. Its file tools are confined to the
  repository and cannot write `.git/` or `.github/`.
- **Credential files stay out of the prompt.** `.env` files (templates such as `.env.example`
  aside), private keys and registry credentials in the checkout are not readable by the builtin
  agent, Claude Code or opencode, and what the tools return is masked. Claude Code and opencode
  cannot open files outside the repository. Codex's shell can read any file of the checkout: keep
  credential files out of the repository and out of earlier steps of the job.
- **Changes are checked before they are committed.** A diff that touches `.github/`,
  `.loopback.yml` or `.vexp/` is rejected. The tree is frozen before verification, so the commit
  contains exactly what was verified and tested.
- **Untrusted input stays data.** Ticket text and logs reach the agent wrapped as untrusted
  content, and secrets are masked out of everything the harness reports.
- **Your tests run in a sandbox.** Installs, tests and the type-check run only inside bubblewrap,
  where they cannot see the action's process or any other, cannot use sudo or Docker, and cannot
  leave anything running for the push (see above).
- **Same-user caveat.** The agent runs as the runner's user. On a self-hosted runner that also
  holds other secrets, prefer the `loopback-runner` container, or a dedicated ephemeral runner.
  Codex runs in its own bubblewrap sandbox, where the action's process is not visible.

## Uploading source maps

Add a step after your production build. The job needs `id-token: write`:

```yaml
      - run: npm run build
      - uses: Loopback-so/action@v1
        with:
          command: upload-sourcemaps
          sourcemaps-path: dist
          url-prefix: /
          release: my-app@${{ github.ref_name }}
```

The action uploads each `.map` file under `sourcemaps-path` with the URL path its script is served
at. It skips `node_modules`, and it skips files that are not version 3 source maps, with a warning.
The embedded sources (`sourcesContent`) are removed before the upload, so your code does not leave
the runner. The API also refuses any map that still carries them.

## Releasing the action

The action runs `dist/index.js` on `node24`. Build it with `pnpm --filter @loopback/action build`,
then commit `dist/` on the release tag, including `dist/package.json` and
`dist/node_modules/vexp-sdk`. That package is shipped as is, not bundled, because its code must
not be rewritten.
