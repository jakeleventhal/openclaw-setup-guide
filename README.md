# OpenClaw Setup Guide

## Provisioning a Hetzner Server

Here we will be provisioning a cloud VPS on Hetzner with 8 vCPUs and 16 GB RAM for ~$36/mo.

* Sign up for an account on [Hetzner](https://accounts.hetzner.com/signUp)
* Go to the [console](https://console.hetzner.com/) and click the "Create Server" button on the default project
* Click Create Resource and select "Servers"
* Their UI is weird so first scroll down to Location and choose the region closest to you then go back to the top
* For Type select "Regular Performance" and CPX41 (8 vCPUs / 16 GB RAM)
* For Image select "Ubuntu"
* For Networking keep the defaults (Public IPv4 and IPv6)
* On your local machine, generate an ssh key by running "ssh-keygen -t ed25519". Copy the contents of the ".pub" file that gets generated in your ~/.ssh folder to your clipboard and add paste the contents in the SSH keys section in Hetzner
* Skip Volumes, Firewalls, and Backups.
* Skip everything else except name and give it a name optionally

## Initial SSH Setup

Before doing anything else, set up a simple alias for SSH since you’ll want to get back to this. Add the following to your SSH config on your personal machine in "~/.ssh/config" for easy access (i.e. "ssh openclaw" — you can name it whatever you want):

```
Host openclaw
  HostName <your ip here>
  User root
  Port 22
  IdentityFile ~/.ssh/id_ed25519

```

Now SSH into your new server ("ssh openclaw"), install zsh, curl, and git

```
sudo apt update && sudo apt install zsh curl git -y
chsh -s $(which zsh)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

```

## OpenClaw Hetzner Guide

Follow [Hetzner setup guide](https://docs.openclaw.ai/install/hetzner#1-provision-the-vps) starting at step 1. Take note of the exceptions below:

### Step 5 (Environment)

Make sure to replace both of the secrets that say to be changed with random output from the openssl command provided, except add one more env var:

* <YOUR_APP_NAME>_READONLY_DATABASE_URL

You can add more as needed if you have multiple apps/services.

### Step 6 (Docker Compose)

Fully delete the existing docker-compose.yml file and replace it with the one provided in the setup guide, but with the following changes:

* Replace "build" with the following

```
build:
  context: .
  args:
    OPENCLAW_DOCKER_APT_PACKAGES: "postgresql-client"

```

* Add the the following line to the environment section:

```
- <YOUR_APP_NAME>_READONLY_DATABASE_URL=${<YOUR_APP_NAME>_READONLY_DATABASE_URL}

```

* Ensure to keep the `--allow-unconfigured` line for now. We will remove it later

### Skip Step 7 (Required Binaries) and Binaries Part of Step 8

This can be done later.

### Step 9 (Verify Gateway)

* You can use the nickname you gave it from the SSH config file earlier instead of "root@<ip>"
* Ensure to wait until the logs appear before trying to access it in your local machine browser
* The directions here are wrong about visiting it in the browser. You need to visit the URL provided, but appended with "?token=<your gateway token you added to .env>"
* You will see "pairing required" in the OpenClaw dashboard
* On the SSH machine, run

```
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js devices list

```

* You will see a UUID request id; copy this, then run the following but replace the UUID at the end:

```
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js approve <UUID>

```

You should now be able to access OpenClaw via your machine’s browser.

## Configuring Chat

Next we need to configure our chat setup with a model provider.

1. From the SSH machine run:

```
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js configure

```

* Select Local
* Select Model and follow the auth flow. This will restart your instance.

2. You should now be able to chat with your model. Say "Hello" and follow the chat prompts.

## Setting Up Telegram

1. From the SSH machine run:

```
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js configure

```

2. Then select Channels and select Telegram. Follow the guide and then message @BotFather on Telegram. You will get a pairing code, with that, run:

```
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js pairing approve telegram <pairing code>

```

3. Message the agent via Telegram to verify it is working. You should be able to message from both the site and via Telegram now.

You can use "slash commands" with your Telegram bot. If you type "/" you will see some options. A good one to run is "/model" to quickly change to a new model like 5.3 Codex Spark, or use "/new" to start a new thread.

## Securing Connections

1. Use the token you added to your .env in the last step:

```
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js config set gateway.mode local
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js config set gateway.bind lan
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js config set gateway.auth.mode token
docker exec -it openclaw-openclaw-gateway-1 node dist/index.js config set gateway.auth.token "<your token here>"

```

2. Then delete the "--allow-unconfigured" line in the docker-compose.yml file, then restart docker-compose instance:

```
docker compose down
docker compose up -d openclaw-gateway

```

3. Verify it works by running the following on your desktop machine:

```
# YOU MUST RUN THIS IF YOU WANT TO VISIT THE WEB VIEWER
ssh -N -L 18789:127.0.0.1:18789 openclaw

```

4. Then visiting: [http://127.0.0.1:18789/openclaw/](http://127.0.0.1:18789/openclaw/) (bookmark this).

## Adding Git

1. Set up SSH for GitHub (be sure to update the last 3 lines with your information):

```
# Create shared SSH folder on host machine
mkdir -p /opt/openclaw/ssh
mkdir -p /opt/openclaw/repos
sudo chown -R 1000:1000 /opt/openclaw/repos
chmod 700 /opt/openclaw/ssh
ssh-keygen -t ed25519 -C "openclaw-vps" -f /opt/openclaw/ssh/id_ed25519_github -N ""
ssh-keyscan github.com >> /opt/openclaw/ssh/known_hosts
chown -R 1000:1000 /opt/openclaw/ssh /opt/openclaw/repos
chmod 700 /opt/openclaw/ssh

cat >/opt/openclaw/ssh/config <<'EOF'
Host github.com
  HostName github.com
  User git
  IdentityFile /home/node/.ssh/id_ed25519_github
  UserKnownHostsFile /home/node/.ssh/known_hosts
  IdentitiesOnly yes
  StrictHostKeyChecking accept-new
EOF

chmod 600 /opt/openclaw/ssh/id_ed25519_github /opt/openclaw/ssh/config
chmod 644 /opt/openclaw/ssh/id_ed25519_github.pub /opt/openclaw/ssh/known_hosts

git config --global user.name "Your Name"
git config --global user.email "Your Email"

```

2. Add the contents of "/opt/openclaw/ssh/id_ed25519_github.pub" to GitHub as an SSH token.

3. Next edit your docker-compose.yml file to add the following volumes:

```
volumes:
  - /opt/openclaw/ssh:/home/node/.ssh:ro
  - /opt/openclaw/repos:/repos

```

4. And add the following under "openclaw-gateway":

```
services:
  openclaw-gateway:
    user: "1000:1000"

```

5. Then restart your docker-compose instance:

```
docker compose down
docker compose up -d openclaw-gateway

```

## Add the Ability to Read from Database

1. On the host machine, clone the repository in the shared "repos" folder we created:

```
cd /opt/openclaw/repos
git clone <your repo URL>

```

2. Verify it is working with the following command:

```
docker exec -it openclaw-openclaw-gateway-1 sh -lc 'cd /repos/<your repo> && git pull'

```

3. Start a new chat with your agent by running "/new" then paste in the following text:

> 1. You have the ability to read GitHub repos under `/repos`. Commit this to memory.
>
> 2. Set up a cron job to run "git pull" in each repo once per day
>
> 3. Under `/repos/<your repo name>` is the repo for <repo name>. Read the root README.md for a brief overview of what it does. Commit this to memory.
>
> 4. Using the repo, along with access to readonly environment variable that you have access to (<YOUR_APP_NAME>_READONLY_DATABASE_URL), you should be able to answer questions using psql by querying our production database. For information about the schema, look to <path to schema code>. All the schema info you need is in there. You should be able to answer query questions accordingly if I ever ask and know how to answer them by making SQL queries to get the information that you need. Commit this to memory.

4. Verify it is working by asking a question that requires information from your database such as:

> "How many sign ups did we get in the last 24 hours for <app name>?"
