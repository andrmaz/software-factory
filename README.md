# Mastra Factory

Mastra Factory is an open source environment for building software with coding agents. Connect your repository to turn issues into plans, implementations, and reviewed pull requests.

Scaffold it with [`npm create factory`](https://www.npmjs.com/package/create-factory). This project contains the Factory Server and its configuration. Keep it separate from the repository you want agents to change.

Read the [documentation](https://factory.mastra.ai/) or [watch the Mastra Factory overview](https://youtu.be/iMA-Xkhj7fU).

## Start the Factory Server

This template uses Mastra platform services for authentication and sandboxes, and PostgreSQL for storage. `npm create factory` provisions a platform project and a Postgres database and writes their credentials to `.env`. For a direct checkout, run `npm install`, copy `.env.example` to `.env`, and fill in the values from your Mastra platform project. To run without the platform, use `npm create factory -- --no-platform`, which scaffolds the self-hosted template instead.

`npm create factory` also generates `FACTORY_CREDENTIAL_ENCRYPTION_KEY`, which encrypts stored model keys and integration secrets. If it's missing from `.env`, generate a key once for this project:

```bash
openssl rand -base64 32
```

Save the output as `FACTORY_CREDENTIAL_ENCRYPTION_KEY` in `.env`. Preserve the key across restarts and deployments, and keep a protected backup. See [credential encryption](https://factory.mastra.ai/reference/configuration#stored-credential-encryption) for details.

From the Factory project directory, start the server:

```bash
npm run dev
```

With the default setup, open the local URL printed by the server and sign in through Mastra platform. One server serves both the Factory UI and API. After login you'll see an onboarding wizard, select the repository agents should change. Use **Manage GitHub connection** to grant the GitHub App access if the repository is missing. Optionally add Linear. Connect a model provider using an API key or a supported subscription, then choose the Factory model.

## Run your first issue

1. Open **Settings → Work Intake → GitHub issues**. Enable **Sync GitHub issues** and select your repository. Each teammate chooses their own issue sources.
1. Create a small GitHub issue, such as adding contribution guidance to the repository's README.
1. Find the issue in **Work → Intake**, select **Investigate**, and open its session to follow the agent's work.

Continue with the [issue-to-pull-request walkthrough](https://factory.mastra.ai/#create-your-first-pull-request) to review a plan and take the change through implementation and pull request review.

## Configure your Factory

Choose authentication, storage, and sandboxes independently. Model providers and issue sources are configured through the Factory UI. Server settings live in `.env`; restart the server after changing them.

| Configuration                                              | What you can change                                                           |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------- |
| [Models](https://factory.mastra.ai/configure/models)       | Provider access, personal or organization credentials, and the default model. |
| [GitHub](https://factory.mastra.ai/configure/github)       | Repository access and personal issue intake.                                  |
| [Linear](https://factory.mastra.ai/configure/linear)       | Workspace connection, project selection, and routing issues to a Factory.     |
| [Jira](https://factory.mastra.ai/configure/jira)           | Site connections, project selection, and routing issues to a Factory.         |
| [Slack](https://factory.mastra.ai/configure/slack)         | App setup, account linking, and starting sessions from Slack.                 |
| [Auth](https://factory.mastra.ai/configure/auth)           | Mastra platform sign-in or another provider with Server and Studio support.   |
| [Storage](https://factory.mastra.ai/configure/storage)     | The database connection or Factory storage adapter.                           |
| [Sandboxes](https://factory.mastra.ai/configure/sandboxes) | Mastra platform, local execution, or another Mastra sandbox provider.         |

The server requires `DATABASE_URL`, a PostgreSQL database with pgvector. Deployments to Mastra platform receive it from the project's attached database.

Slack is bundled but disabled unless you set Slack credentials. For webhook mode, set `SLACK_APP_SIGNING_SECRET`, `SLACK_APP_BOT_TOKEN`, `SLACK_APP_CLIENT_ID`, and `SLACK_APP_CLIENT_SECRET`. For Socket Mode, use `SLACK_APP_TOKEN` instead of `SLACK_APP_SIGNING_SECRET`; the other three variables are still needed. A bot token alone cannot receive events or link Slack users, and an incomplete configuration fails at startup rather than silently disabling Slack. Set `MASTRACODE_PUBLIC_URL` to your HTTPS Factory origin for account-linking callbacks; for local development, set `MASTRACODE_CHANNELS_PUBLIC_URL` to an HTTPS tunnel origin instead. See [Slack configuration](https://factory.mastra.ai/configure/slack) for app setup and callback URLs.

With `MASTRA_PLATFORM_SECRET_KEY` (or `MASTRA_PLATFORM_ACCESS_TOKEN`) set, `npm run dev` runs agent commands in local sandboxes on your machine, and deployments use Mastra platform sandboxes, which also need `MASTRA_PROJECT_ID` and `MASTRA_ENVIRONMENT_ID`. To pick a sandbox yourself, pass `sandbox` to `MastraFactory` in `src/mastra/index.ts`, for example `new LocalFactorySandbox()` from `@mastra/factory`.

## Deploy

### Mastra platform

The committed `.mastra-project.json` selects the platform project. The default
deployment environment is `production`. Before deploying:

- Use the Node.js version required by `package.json`, install dependencies with
  `pnpm install --frozen-lockfile`. The project includes the Mastra CLI, so no
  global installation is needed. Run `pnpm run check` before deploying.
- Run `pnpm exec mastra auth whoami` to verify CLI authentication, or
  `pnpm exec mastra auth login` to sign in.
- Confirm production has a ready, attached PostgreSQL database with pgvector.
  The platform injects its `DATABASE_URL`; don't overwrite it with an empty value.
- Set `FACTORY_CREDENTIAL_ENCRYPTION_KEY` in the platform's environment settings.
  Reuse the key that encrypted existing database credentials, not a newly generated
  replacement. Keep it stable and backed up.
- Configure runtime platform credentials (`MASTRA_PLATFORM_ACCESS_TOKEN` or
  `MASTRA_PLATFORM_SECRET_KEY`), `MASTRA_PROJECT_ID`, and `MASTRA_ENVIRONMENT_ID`
  for platform sandboxes and integrations. CLI login is not a substitute for these
  server-side credentials.
- Set `MASTRA_ORGANIZATION_ID` to the owning organization and
  `MASTRACODE_PUBLIC_URL` to the actual HTTPS Factory origin supplied by the
  platform, not the Studio origin or localhost.

Deploy using the project's script:

```bash
pnpm run deploy --env production --workers in-process
```

This targets production with in-process workers. For other environments or
dedicated workers, use `pnpm run deploy --env <name> --workers <mode>`.
Use `--workers dedicated` only when the environment has Redis.

The CLI reports the deployed Factory and Studio URLs. Local env files can overwrite
platform settings during deployment, including secrets; review the selected file
and never upload `.env.example` as production configuration. Keep secrets in the
platform's environment settings or ignored `.env.*` files.

**Authentication:** Keep `MASTRA_PLATFORM_STUDIO_AUTH` unset so hosted Studio
requires platform organization membership. Studio protection alone does not secure
API routes. This project's `factory.prepare()` supplies both `server.auth` and
`studio.auth` using Factory's platform provider; the exported Mastra instance
preserves that configuration. Do not replace it with `auth: null`.

After deployment, visit both reported URLs in a signed-out browser and confirm
sign-in is required. Unauthenticated requests to `/api/agents` and protected
Factory endpoints must be rejected (401/403 or a sign-in redirect), not return
application data. Sign in as an organization member, verify access, then sign out
and confirm access is revoked. Confirm callbacks return to the deployed Factory
and an investigation uses a platform sandbox.

See [Factory deployment](https://factory.mastra.ai/deployment),
[Studio platform authentication](https://mastra.ai/docs/mastra-platform/deploy/studio#authentication),
and [Factory authentication](https://factory.mastra.ai/configure/auth).

### Self-host

Run Factory as a persistent Node.js service on a virtual machine or in a container. Configure persistent storage and set `MASTRACODE_PUBLIC_URL` to the public HTTPS origin. Supply your provider credentials through the deployment environment, then build and start the server:

```bash
npm run build
npm run start
```

Keep the project's CLI dependencies installed for these commands. You can use Mastra platform services while hosting the Factory Server elsewhere. See [self-hosting](https://factory.mastra.ai/deployment#self-host) for runtime requirements and setup steps.

## Scripts

| Script           | What it does                                         |
| ---------------- | ---------------------------------------------------- |
| `npm run dev`    | Start the local Factory Server with its UI and API.  |
| `npm run check`  | Typecheck the Factory Server.                        |
| `npm run build`  | Build the server and Factory UI in `.mastra/output`. |
| `npm run start`  | Run the production build.                            |
| `pnpm run deploy` | Build and deploy to Mastra platform; accepts CLI flags. |

## Troubleshooting

- **Runtime:** Use a Node.js version that matches `engines.node` in this project's `package.json`.
- **Port changes:** Set `PORT` and update `MASTRACODE_PUBLIC_URL` to match. Update callback URLs for any auth or integration apps you manage.
- **Missing issues:** Check repository access and your **Work Intake** selections. See [intake troubleshooting](https://factory.mastra.ai/troubleshooting#intake-is-empty).

See [Troubleshooting](https://factory.mastra.ai/troubleshooting) for sign-in, provider access, and sandbox errors, or [Environment variables](https://factory.mastra.ai/reference/configuration) for server settings.

## License

Apache-2.0
