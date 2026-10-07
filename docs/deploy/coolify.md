---
title: Coolify via GitHub
summary: Beginner-friendly Paperclip deployment on Coolify from a GitHub repository
---

This guide walks you through deploying Paperclip on your own server with [Coolify](https://coolify.io), using a GitHub repository as the source. You do not need to know Docker or run any build commands.

**What you will have at the end:** a private Paperclip website on your own domain, with a database, secure HTTPS, and automatic secret generation. You will log in as the first admin and can then hire AI agents.

**Time needed:** about 20 minutes, plus server setup.

## Before you start

Check that you have:

- [ ] **A server with Coolify installed.** If you do not have this yet, follow Coolify's [self-hosted install guide](https://coolify.io/docs/start-with-self-hosted) first. Its proxy must show **Running** on the server page.
- [ ] **A GitHub account** (free).
- [ ] **Enough server memory.** 4 GB RAM is a comfortable minimum for Paperclip plus its database. 2 GB works for a first look, but agent runs are happier with 4 GB or more.
- [ ] **Optional: your own domain name.** You can use a free test address that Coolify generates, but a real domain is what you want long-term.
- [ ] **Optional: an AI provider key** (for example an Anthropic or OpenAI API key). You can add this later from inside Paperclip.

### Two words you will see

- **Repository ("repo")**: your copy of the Paperclip code on GitHub. Coolify reads it from there.
- **Compose file**: a settings file in the repo that tells Coolify exactly what to run. You pick it; you do not edit it.

## Step 1 — Get the Paperclip code into your GitHub

1. Open [github.com/paperclipai/paperclip](https://github.com/paperclipai/paperclip).
2. Click **Fork**, then **Create fork**. You now have your own copy.

> **Important:** your fork must contain the file `docker/docker-compose.coolify.yml`. If you forked an older copy, open your fork on GitHub and click **Sync fork → Update branch** so the file appears. (If you maintain your own copy of the repo, push your branch instead.)

## Step 2 — Create the application in Coolify

1. In Coolify, open **Projects**, click **+ New Project**, name it `paperclip`, and click **Continue**.
2. Click **+ New Resource**, then choose **Public Repository**.
3. Fill in the fields exactly like this:

   | Field | Value |
   | --- | --- |
   | Repository URL | `https://github.com/YOUR-USERNAME/paperclip` |
   | Branch | `master` (or your branch name) |
   | Build Pack | **Docker Compose** |
   | Base Directory | `docker` |
   | Docker Compose Location | `docker-compose.coolify.yml` |

4. Click **Check Repository**, then **Continue** / **Save**.

This file pulls Paperclip's official prebuilt image, so there is no long build step and it works on small servers.

## Step 3 — Add your website address

1. Open the new resource, then the **Domains** tab.
2. Click **Add domain**.
3. Choose the **paperclip** service and enter the internal port `3100`.
4. For your first test you can keep the free address Coolify generates (it looks like `https://something.your-server-ip.sslip.io`). Later, type your own domain to replace it.
5. Copy the exact address shown — you need it in the next step. It must start with `https://`.

## Step 4 — Set one environment variable

Open **Environment Variables** and add:

| Name | Value |
| --- | --- |
| `PAPERCLIP_PUBLIC_URL` | The exact address from Step 3, with **no** trailing slash and **no** `:3100` |

For example: `https://paperclip.example.com`.

Everything else required is generated for you: Coolify creates the database password and the two signing secrets on the first deploy. Do not change or delete variables whose names start with `SERVICE_` — those are the generated secrets.

Optional, add now or later:

| Name | Value |
| --- | --- |
| `ANTHROPIC_API_KEY` | Your Anthropic API key, if you have one |
| `OPENAI_API_KEY` | Your OpenAI API key, if you have one |
| `PAPERCLIP_TELEMETRY_DISABLED` | `1` to opt out of anonymous usage statistics |

## Step 5 — Deploy

1. Click **Deploy** and watch the deployment log.
2. Wait for the deployment to finish (usually a few minutes — it is downloading the Paperclip image).
3. Open the address from Step 3 in your browser.
4. Check that `https://your-address/api/health` opens and shows the server is healthy.

If the page says **No available server** right after deploying, wait two minutes and reload — Paperclip needs a moment to start its database and finish initial setup.

## Step 6 — Become the instance admin (one command)

Paperclip is private by default: nobody can use it until an admin exists. Because this is a public deployment, create the first admin from Coolify's terminal:

1. In Coolify, open your Paperclip resource and click **Terminal**.
2. Type this command and press Enter:

   ```sh
   cd /app && pnpm paperclipai auth bootstrap-ceo
   ```

3. The command prints a line starting with **Invite URL:**. Copy that full link and open it in your browser.
4. Create your account (email and password). You are now the instance admin.

The invite link expires after 72 hours. If the command says *"Instance already has an admin user"*, someone already claimed the instance — just sign in normally.

## Step 7 — Your first run

1. Open **Apps → Connections** to connect an AI provider (an API key or a subscription sign-in). The Docker image already includes the Claude, Codex, OpenCode, Gemini, and Kimi command-line tools.
2. Create your first company, then follow the [five-minute getting-started path](https://docs.paperclip.ing/guides/getting-started/five-minute-path/) to hire your first agent and give it a task.

## Updating Paperclip later

- **Redeploy** in Coolify recreates the containers from the same version.
- **Upgrade**: open `docker/docker-compose.coolify.yml` in your GitHub repo and change the image line to a newer tag, for example from `ghcr.io/paperclipai/paperclip:latest` to a specific release. Commit the change, then click **Redeploy** in Coolify. Redeploys restart agent work in progress, so do it when things are quiet.

## Backups

Enable scheduled backups in Coolify for both storage volumes and keep a copy off the server:

| Volume | What it holds |
| --- | --- |
| `paperclip-data` (mounted at `/paperclip`) | Uploads, agent workspaces, the local encryption key |
| `paperclip-pgdata` | The PostgreSQL database |

Back up both together. If you restore the database without the `/paperclip` volume, encrypted secrets cannot be read.

## Troubleshooting

| What you see | What to do |
| --- | --- |
| **No available server** | Wait two minutes and reload. Then confirm the domain includes `:3100` and that the health check uses port `3100` and path `/api/health`. |
| **Site not secure / certificate error** | Give the certificate a minute to issue. Confirm the domain points at your server (for `sslip.io` test addresses, no setup is needed). |
| **Login works but you keep being sent back to the login page** | `PAPERCLIP_PUBLIC_URL` must match the browser address exactly, including `https://` and no trailing slash. Fix it and redeploy. |
| **Deployment fails immediately** | Open the deployment log. The most common cause is a wrong Base Directory or Compose Location — they must be `docker` and `docker-compose.coolify.yml`. |
| **Agent replies with an authentication error** | Add a provider key under **Apps → Connections** (or set `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` and redeploy). |
| **I need to change my domain** | Add the new domain, update `PAPERCLIP_PUBLIC_URL` to match, then redeploy. |
| **Everything is healthy but the page is blank** | Hard-refresh the browser (or try a private window) to clear a cached old bundle. |

For deeper issues, see the general [Docker guide](docker.md) and Coolify's [no available server troubleshooting](https://coolify.io/docs/troubleshoot/applications/no-available-server).

## If this path does not fit

- **No GitHub account?** Create the resource with **Docker Compose Empty** and paste the contents of `docker/docker-compose.coolify.yml` into the editor. Steps 3–7 are unchanged.
- **Want Coolify to build the code from your repo instead of using the prebuilt image?** Use `docker-compose.coolify-source.yml` (same Base Directory and Build Pack settings). This build compiles native code and needs a server with about 4 vCPU / 8 GB RAM; give it time. Enable **Advanced → Build → Source commit availability → Available during build** so the version stamp is filled in.
- **Want a single container without a separate database?** Use **New Resource → Docker Image** with `ghcr.io/paperclipai/paperclip:latest`, port `3100`, a volume at `/paperclip`, and the same environment variables plus `BETTER_AUTH_SECRET` (generate a long random value) and `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET`. Paperclip then runs an embedded database inside the container. The Compose path above is recommended because the database is easier to back up and upgrade separately.
