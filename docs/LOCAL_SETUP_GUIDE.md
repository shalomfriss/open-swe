# Complete local setup, with provider screenshots

This guide sets up Open SWE on one computer. The dashboard and backend run at **http://localhost:2024**; Managed Deep Agents (MDA), which runs every agent thread, listens at **http://localhost:2025**. PostgreSQL stores Open SWE's data. A restricted ngrok tunnel gives GitHub and Slack a stable public webhook address. The coding sandboxes themselves use LangSmith.

The screenshots below come from the providers' own documentation. They show where to look, but a provider may rename or rearrange a control. The linked provider pages are the current reference. Never put a real key, token, secret, or private key in a screenshot or commit it to Git.

## Before you begin

You need:

| Item | Why |
|---|---|
| Python 3.14+ and [uv](https://docs.astral.sh/uv/) | Backend and MDA Python environments |
| Node 22.22.2+ and [pnpm](https://pnpm.io/installation) | Dashboard build |
| Rust and Cargo ([rustup](https://rustup.rs/)) | Build the vendored MDA CLI |
| Docker Desktop with a running daemon, **or** a PostgreSQL 16 server | Local database |
| [ngrok account and agent](https://ngrok.com/download) | Stable HTTPS hostname for webhooks |
| [GitHub account](https://github.com/) allowed to create and install an App | GitHub login and repository access |
| [Slack workspace](https://slack.com/) where you may create an app | Slack bot and events |
| [LangSmith account](https://smith.langchain.com/) with sandbox access | Traces and coding sandboxes |
| [OpenAI API account](https://platform.openai.com/) or another supported model provider | Model calls |

Check local tools before continuing:

```bash
python3 --version
uv --version
node --version
pnpm --version
cargo --version
docker info
ngrok version
```

If you use a local PostgreSQL server instead of Docker, `docker info` is optional.

## 1. Install the project

```bash
git clone https://github.com/langchain-ai/open-swe.git
cd open-swe
uv venv
source .venv/bin/activate
uv sync --all-extras
```

If you already have this checkout, run the commands from `uv venv` onward in the repository root. Later, `make build-dashboard` installs the JavaScript workspace dependencies and builds the UI.

## 2. Claim a stable ngrok domain

1. Open the [ngrok dashboard](https://dashboard.ngrok.com/), create an account, and install the agent.
2. Find **Your Authtoken** in the dashboard and run the exact `ngrok config add-authtoken ...` command it shows. Treat that token as a secret.
3. Go to **Gateway → Domains** and find the free static dev domain already assigned to your account. Write down the hostname, such as `your-name.ngrok-free.dev`, without `https://`. The [ngrok domain guide](https://ngrok.com/blog/free-static-domains-ngrok-users) describes the current flow.

![ngrok dashboard authtoken page, with the token obscured](images/local-setup/ngrok-authtoken.png)

*Official [ngrok authtoken screenshot](https://ngrok.com/blog/go-crud-app-1); the dashboard's navigation may now differ.*

Keep this hostname: use the **same** hostname in the GitHub App, Slack manifest, `.env`, and `make tunnel`. Do not accept an automatically changing hostname on each restart.

## 3. Create and install a GitHub App

1. Open [GitHub Apps settings](https://github.com/settings/apps), or navigate through **Profile picture → Settings → Developer settings → GitHub Apps → New GitHub App**. The [GitHub registration guide](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app) explains the form.
2. Give the App a unique name, for example `open-swe-yourname-local`. Set its **Homepage URL** to `http://localhost:2024`.
3. Set **Callback URL** to `http://localhost:2024/dashboard/api/auth/callback`.
4. Turn off **Request user authorization (OAuth) during installation**.
5. Generate a webhook secret with `openssl rand -hex 32`. Store the output privately. Set **Webhook URL** to `https://your-name.ngrok-free.dev/webhooks/github`, set **Webhook secret** to that output, and leave the webhook active. A stable domain is required for deliveries to reach the next local session.
6. Set the permissions and events in the table below, then click **Create GitHub App**.

| Section | Setting |
|---|---|
| Repository permissions | Contents: **Read & write**; Pull requests: **Read & write**; Issues: **Read & write**; Checks: **Read & write**; Commit statuses: **Read-only**; Workflows: **Read & write**; Metadata: **Read-only** |
| Optional repository permissions | Code scanning alerts: **Read-only**; Actions: **Read-only** for CI diagnostics, or **Read & write** only if you want `/baby-sit` to rerun GitHub Actions jobs |
| Organization permissions | Members: **Read-only** if you will use `ALLOWED_GITHUB_ORGS` |
| Subscribe to events | Issue comment; Pull request review; Pull request review comment; Check run; Check suite; Workflow run; optionally Status for legacy commit statuses |

These are the permissions and events specified in the repository's [installation guide](INSTALLATION.md#3-create-a-github-app). Limit the App installation to repositories that every admitted local user is trusted to access.

After creating the App, copy these values from its settings:

| GitHub location | `.env` variable |
|---|---|
| General → **App ID** | `GITHUB_APP_ID` |
| General → **Client ID** (starts with `Iv`) | `GITHUB_APP_CLIENT_ID` |
| **Client secrets → Generate a new client secret** | `GITHUB_APP_CLIENT_SECRET` |
| **Private keys** or **Credentials → Key pairs → New key**; downloads a `.pem` | `GITHUB_APP_PRIVATE_KEY` |
| The webhook secret you generated above | `GITHUB_WEBHOOK_SECRET` |

![GitHub App credentials page showing the Key pairs section and New key button](images/local-setup/github-key.png)

*Official [GitHub private-key screenshot and instructions](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps). Keep the downloaded `.pem` outside the repository.*

Then choose **Install App** in its sidebar, select your account or organization, and grant access to the repositories you want Open SWE to use. The numeric suffix of the installation page URL is `GITHUB_APP_INSTALLATION_ID`; it is **different from App ID**. See [GitHub's installation instructions](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app).

The `.pem` must be a real private key downloaded from this App. It should start with `-----BEGIN RSA PRIVATE KEY-----` or `-----BEGIN PRIVATE KEY-----` and have a matching end line. In `.env`, put the whole PEM on **one double-quoted line**, replacing actual line breaks with literal `\n` sequences. Do not paste a file path or an abbreviated key. If key parsing fails, download a new key and rebuild this value from the complete file.

## 4. Get a LangSmith key and a model key

In [LangSmith](https://smith.langchain.com/), open **Settings → API Keys → Create API Key** and save the key when it is shown. Use a key with access to the workspace where you will run Open SWE; a tracing-only key is insufficient because Open SWE also uses LangSmith sandboxes and trace links. See the [LangSmith key guide](https://docs.langchain.com/langsmith/create-account-api-key). Set the same `LANGSMITH_API_KEY` in both `.env` files.

For the simplest model setup, open the [OpenAI API platform](https://platform.openai.com/), select your organization and project, go to **API Keys**, and choose **Create new secret key**. Save it as `OPENAI_API_KEY` in both `.env` files. The [OpenAI key guide](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key) and [project key guide](https://help.openai.com/en/articles/9186755-managing-projects-in-the-api-platform) cover this flow. An Anthropic key can be used as `ANTHROPIC_API_KEY` instead; the [model provider section](INSTALLATION.md#4-model-providers-and-api-keys) lists the supported providers.

![OpenAI project API Keys page with Create new secret key highlighted](images/local-setup/openai-key.png)

*Official [OpenAI project-key screenshot](https://help.openai.com/en/articles/9186755-managing-projects-in-the-api-platform). The new key is shown only when created, so save it privately then.*

## 5. Create the Slack app

1. Open [Your Slack Apps](https://api.slack.com/apps), choose **Create New App → From an app manifest**, and select your workspace. See [Slack's manifest instructions](https://docs.slack.dev/app-manifests/configuring-apps-with-app-manifests/).
2. In the **JSON** tab, paste the complete manifest from [Create the Slack app](INSTALLATION.md#5-create-the-slack-app). Replace **every** `<your-url>` with your ngrok **hostname only**, such as `your-name.ngrok-free.dev`; the manifest already includes `https://`. Give the app and bot a distinct local name if a shared Open SWE bot already exists in the workspace.
3. Review and create the app. Its manifest configures Events, Interactivity, `/oswe`, OAuth scopes, and webhook URLs together. If Slack cannot verify the Events Request URL yet, finish this setup, start the backend and tunnel, then go to **Event Subscriptions → Retry**.
4. Open **Install App** and install it to the workspace. In **OAuth & Permissions**, copy the **Bot User OAuth Token** beginning with `xoxb-` into `SLACK_BOT_TOKEN`.
5. Open **Basic Information → App Credentials**. Copy **App ID** into `SLACK_APP_ID` and **Signing Secret** into `SLACK_SIGNING_SECRET`. Slack's [request-verification guide](https://docs.slack.dev/authentication/verifying-requests-from-slack/) identifies this signing secret.
6. In Slack, open the bot's profile, choose **⋮ → Copy member ID**, and put it in `SLACK_BOT_USER_ID`. Put the bot's handle, without `@`, in `SLACK_BOT_USERNAME`.

![Slack Create an app dialog showing From an app manifest](images/local-setup/slack-manifest.png)

*Official [Slack manifest creation screenshot](https://docs.slack.dev/tools/bolt-js/tutorials/custom-steps-workflow-builder-new/).*

![Slack Install App page showing Bot User OAuth Token](images/local-setup/slack-token.png)

*Official [Slack bot-token screenshot](https://docs.slack.dev/tools/python-slack-sdk/tutorial/uploading-files/); the illustrated token is obscured.*

![Slack Basic Information page showing App Credentials](images/local-setup/slack-basic.png)

*Official [Slack Basic Information screenshot](https://docs.slack.dev/authentication/verifying-requests-from-slack/).*

![Slack Signing Secret control in App Credentials](images/local-setup/slack-secret.png)

*Official [Slack signing-secret screenshot](https://docs.slack.dev/authentication/verifying-requests-from-slack/); the illustrated secret is obscured.*

For local Slack account linking, the manifest also includes `https://your-name.ngrok-free.dev/dashboard/api/slack/callback`. The stock `make tunnel` policy permits only `/webhooks/*`, so that optional OAuth callback requires a local ngrok redirect policy that sends the callback to `http://localhost:2024/dashboard/api/slack/callback` **with its full query string**. Keep dashboard pages and APIs private. The [development guide](DEVELOPMENT.md#3-tunnel-for-webhooks) describes the required relay behavior.

## 6. Write the two environment files

Create `.env` in the repository root. Replace each `REPLACE_...` placeholder with a real value. Use your own GitHub login for both `ALLOWED_GITHUB_USERS` and `CONFIGURED_ADMINS`. Do not commit this file.

```dotenv
LANGSMITH_API_KEY="REPLACE_LANGSMITH_KEY"
LANGSMITH_TRACING="true"
LANGSMITH_PROJECT="open-swe-local"
OPENAI_API_KEY="REPLACE_OPENAI_KEY"

GITHUB_APP_ID="REPLACE_GITHUB_APP_ID"
GITHUB_APP_CLIENT_ID="REPLACE_GITHUB_CLIENT_ID"
GITHUB_APP_CLIENT_SECRET="REPLACE_GITHUB_CLIENT_SECRET"
GITHUB_APP_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\nREPLACE_ALL_BASE64_LINES_JOINED_WITH_LITERAL_BACKSLASH_N\n-----END RSA PRIVATE KEY-----"
GITHUB_WEBHOOK_SECRET="REPLACE_GITHUB_WEBHOOK_SECRET"
GITHUB_APP_INSTALLATION_ID="REPLACE_GITHUB_INSTALLATION_ID"
ALLOWED_GITHUB_USERS="REPLACE_YOUR_GITHUB_LOGIN"

SLACK_BOT_TOKEN="REPLACE_XOXB_BOT_TOKEN"
SLACK_SIGNING_SECRET="REPLACE_SLACK_SIGNING_SECRET"
SLACK_APP_ID="REPLACE_SLACK_APP_ID"
SLACK_BOT_USER_ID="REPLACE_SLACK_BOT_MEMBER_ID"
SLACK_BOT_USERNAME="REPLACE_SLACK_BOT_HANDLE"
SLACK_PUBLIC_BASE_URL="https://your-name.ngrok-free.dev"

TOKEN_ENCRYPTION_KEY="REPLACE_RANDOM_BASE64_VALUE"
DASHBOARD_JWT_SECRET="REPLACE_RANDOM_HEX_VALUE"
CONFIGURED_ADMINS="REPLACE_YOUR_GITHUB_LOGIN"

MDA_RUNTIME_URL="http://localhost:2025"
REMOTE_RUNTIME_TOKEN_SECRET="REPLACE_RANDOM_HEX_VALUE"
RUN_COMPLETE_WEBHOOK_SECRET="REPLACE_ANOTHER_RANDOM_HEX_VALUE"
COMPLETION_WEBHOOK_URL="https://your-name.ngrok-free.dev/webhooks/run-complete"
```

Generate the four unrelated random values with separate commands:

```bash
openssl rand -base64 32   # TOKEN_ENCRYPTION_KEY
openssl rand -hex 32      # DASHBOARD_JWT_SECRET
openssl rand -hex 32      # REMOTE_RUNTIME_TOKEN_SECRET
openssl rand -hex 32      # RUN_COMPLETE_WEBHOOK_SECRET
```

If your GitHub PEM says `BEGIN PRIVATE KEY` rather than `BEGIN RSA PRIVATE KEY`, preserve its **actual** begin/end lines. To convert a downloaded PEM to the expected single-line `.env` value, run `python3 -c 'from pathlib import Path; print(Path("/absolute/path/to/downloaded-key.pem").read_text().strip().replace("\n", "\\n"))'`; this prints the private key to your terminal, so avoid screen sharing and clear that terminal afterward. Use the exact output between one pair of double quotes.

If you use Docker's default database, **omit** `POSTGRES_URI` entirely: `make dev` starts the `postgres:16` container on `127.0.0.1:5433`. If you use an existing PostgreSQL server, add `POSTGRES_URI="postgresql://USER:PASSWORD@127.0.0.1:PORT/postgres"` (omit `:PASSWORD` if local authentication does not require it). PostgreSQL must be running before `make dev`.

Create `mda/.env` separately. It needs its **own** copy of the LangSmith and model keys:

```dotenv
OPEN_SWE_BACKEND_URL="http://localhost:2024"
MDA_DEPLOYMENT_NAME="open-swe-your-name-local"
LANGSMITH_API_KEY="REPLACE_SAME_LANGSMITH_KEY"
OPENAI_API_KEY="REPLACE_SAME_OPENAI_KEY"
```

Give `MDA_DEPLOYMENT_NAME` a stable name unique to your machine. Setting it explicitly avoids a blocking-call error in MDA's local browser middleware. If you chose Anthropic, use `ANTHROPIC_API_KEY` in both files in place of `OPENAI_API_KEY`. Keep both files local and ignored by Git. Local `LANGGRAPH_URL`, `DASHBOARD_BASE_URL`, and `DASHBOARD_API_BASE_URL` can retain their localhost defaults; do not point them at ngrok.

MDA's dev server refuses loopback completion webhooks. Keep the restricted ngrok tunnel running for dashboard and Desktop tasks as well as Slack and GitHub. After changing either `.env` file, restart its corresponding service.

Check the GitHub installation ID and private key locally without displaying the key:

```bash
./.venv/bin/python - <<'PY'
from dotenv import dotenv_values
from cryptography.hazmat.primitives.serialization import load_pem_private_key

settings = dotenv_values('.env')
assert settings.get('GITHUB_APP_INSTALLATION_ID'), 'Set GITHUB_APP_INSTALLATION_ID'
load_pem_private_key(settings['GITHUB_APP_PRIVATE_KEY'].encode(), password=None)
print('GitHub installation ID is set and private key parses')
PY
```

## 7. Build and start everything

From the repository root, run the dashboard build once (and again after pulling UI changes):

```bash
make build-dashboard
```

Then keep **three terminals** running from the same repository root:

| Terminal | Command | What it starts |
|---|---|---|
| 1 | `make dev` | PostgreSQL Docker container if needed, backend and dashboard at `http://localhost:2024` |
| 2 | `make dev-mda` | MDA runtime at `http://localhost:2025`; first build can take time |
| 3 | `make tunnel NGROK_DOMAIN=your-name.ngrok-free.dev` | Restricted public webhook tunnel to port 2024 |

Start `make dev` first, then MDA and the tunnel. `make tunnel` uses [the checked-in webhook-only traffic policy](../examples/ngrok/webhooks-only.yml), which blocks access to the unauthenticated LangGraph API through the public hostname. The [ngrok traffic policy guide](https://ngrok.com/docs/gateway/traffic-policy) explains that control. **Do not** replace it with unrestricted `ngrok http 2024`.

Use `make dev`, not `make run`: `make run` only serves the FastAPI app on port 8000 and cannot create agent runs. A running `make dev` without `make build-dashboard` or `make dev-ui` may answer `/ok` while showing no dashboard at `/`.

## 8. Verify the whole setup

1. In a fourth terminal, run `curl -i http://localhost:2024/ok`; expect HTTP 200. Open **http://localhost:2024** in a browser and confirm the dashboard loads.
2. Click **Sign in with GitHub**. A successful callback returns you to the dashboard. If the GitHub App OAuth flow fails, compare the callback URL character for character and check `ALLOWED_GITHUB_USERS`.
3. Run `curl -i https://your-name.ngrok-free.dev/webhooks/slack`; it should reach the backend. Run `curl -i https://your-name.ngrok-free.dev/ok`; the tunnel policy should block it with ngrok's 404. The latter must **not** return the backend's `/ok` response.
4. In the Slack App's **Event Subscriptions**, press **Retry** until the Request URL verifies. Invite the bot to a channel and mention it. Inspect incoming requests at **http://localhost:4040**.
5. In GitHub, open your App's **Advanced → Recent Deliveries** and confirm a webhook is delivered after you mention the bot on an issue or PR in an installed repository. The commenting GitHub user should first have signed in to the local dashboard.
6. In the dashboard, open **Admin → Defaults**, set a default repository, and start a simple task. Confirm a run appears in the dashboard and LangSmith. This checks the backend, MDA, GitHub installation, model key, and sandbox access together.

## If something is missing

| Symptom | Check |
|---|---|
| `node --version` fails with a missing Homebrew `.dylib` | Repair the local Node installation before `make build-dashboard`; use a working Node 22.22.2+ binary and verify `node --version` and `pnpm --version` first. |
| Nothing at `localhost:2024` | Keep `make dev` running; check its startup output, port conflicts, and the dashboard build. Open `http://localhost:2024`, not the ngrok hostname. |
| `/ok` works but `/` is blank or missing | Run `make build-dashboard`; then restart `make dev`. |
| Backend refuses to start | Fill every required root `.env` value, especially `MDA_RUNTIME_URL`, `REMOTE_RUNTIME_TOKEN_SECRET`, all five Slack fields, and a GitHub login allowlist. Check PostgreSQL. |
| GitHub App token unavailable or private-key error | Confirm `GITHUB_APP_INSTALLATION_ID` is the installation URL's number. Replace an incomplete or malformed `GITHUB_APP_PRIVATE_KEY` with the complete PEM in single-line `\n` format. |
| GitHub sign-in fails | Check the exact localhost callback URL, Client ID and secret, and `ALLOWED_GITHUB_USERS`. |
| Slack Request URL fails verification | Start `make dev` and `make tunnel`, confirm the manifest uses the same hostname, then click **Retry**. Restart the backend if you changed `SLACK_SIGNING_SECRET`. |
| Dashboard starts a thread but no agent work happens | Keep `make dev-mda` running on port 2025; verify `mda/.env` has `OPEN_SWE_BACKEND_URL`, `LANGSMITH_API_KEY`, and the model key. |
| A task stays on **Working** after the run fails | Check the MDA run error, set `MDA_DEPLOYMENT_NAME` in `mda/.env`, and set `RUN_COMPLETE_WEBHOOK_SECRET` plus the public `COMPLETION_WEBHOOK_URL` in the root `.env`. Keep the restricted tunnel running and restart both services. Start a new task after restarting `make dev-mda`; its dev checkpoints are cleared on restart. |
| `SandboxAuthenticationError: Sandbox feature is not enabled for this organization` | The LangSmith organization associated with `LANGSMITH_API_KEY` cannot create sandboxes. Confirm both `.env` files use a key from the intended organization, then [contact LangChain support](https://support.langchain.com/) with the exact error and ask them to enable or diagnose LangSmith Sandboxes for that organization. Open SWE's normal Slack threads require LangSmith sandboxes. |
| Docker database cannot start | Check Docker Desktop and the `open-swe-postgres` volume. You can point `POSTGRES_URI` at a working PostgreSQL 16 instance instead. |

For further details, use the repository's [local development guide](DEVELOPMENT.md), [full installation reference](INSTALLATION.md), and [MDA runtime guide](../mda/README.md#run-locally).

## Running a task without a LangSmith sandbox

For a task you start yourself, [Open SWE Desktop](../desktop/README.md#install-on-macos) offers **This Mac** in the new-task composer. It runs the agent's file and shell operations in a selected local project through a bridge, so that task does not create a LangSmith sandbox. Keep the desktop app open while the task runs. The backend and MDA still run as above.

1. Install the Desktop app with `brew install --cask langchain-ai/tap/open-swe-desktop`, or follow its [source-build instructions](../desktop/README.md#install-on-macos) after repairing Node.
2. Connect it to `http://localhost:2024` and sign in.
3. Add a local project, start a new task, choose **This Mac**, and select the project. Choose **New worktree** if you want the task in its own Git worktree.

If Desktop says **“Open SWE could not start”** and its terminal log says **“Desktop UI archive checksum mismatch”**, the local backend's generated `ui/.output/desktop/descriptor.json` may name the checksum of your local build while Desktop downloads the published GitHub release for the same commit. With an existing local UI build, quit Desktop and launch it against the local archive instead:

```bash
OPEN_SWE_DESKTOP_UI_ARCHIVE="$PWD/ui/.output/desktop/ui.json.gz" \
  "/Applications/Open SWE.app/Contents/MacOS/Open SWE" \
  --backend-url=http://localhost:2024
```

The [Desktop development instructions](../desktop/README.md#how-it-connects) describe this archive override. Alternatively, when that commit has a published release, replace the generated descriptor with the release's `descriptor.json` so the checksum matches the public archive. A subsequent `make build-dashboard` can regenerate the local descriptor.

This does not change Slack's behavior. A Slack mention starts a normal public thread, which currently needs a LangSmith sandbox; the existing machine bridge only accepts a private thread owned by the person moving it. There is no `SANDBOX_TYPE=local` setting in this checkout. Supporting local execution for Slack-triggered threads would require a code change and an explicit way to authorize commands that Slack users could cause to run on your computer.
