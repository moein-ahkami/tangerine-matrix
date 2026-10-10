# 🍊 Tangerine

**A self-hosted private messaging server built with Matrix and
Synapse.** 🍊🔐

Tangerine is a self-hosted messaging project based on
[Matrix](https://matrix.org/) and
[Synapse](https://github.com/element-hq/synapse), designed for private
communication between trusted users. 💬

The current setup runs Synapse on Arch Linux and manages it with Docker
Compose. Remote access from other devices is provided through Tailscale
and Tailscale Serve.

## Table of Contents

-   [Project Goals](#project-goals)
-   [Architecture](#architecture)
-   [Prerequisites](#prerequisites)
-   [Project Structure](#project-structure)
-   [Install Docker](#install-docker)
-   [Generate Synapse Configuration](#generate-synapse-configuration)
-   [Configure Docker Compose](#configure-docker-compose)
-   [Start the Server](#start-the-server)
-   [Manage User Accounts](#manage-user-accounts)
-   [Configure Tailscale](#configure-tailscale)
-   [Connect a Matrix Client](#connect-a-matrix-client)
-   [Health Checks](#health-checks)
-   [Service Management](#service-management)
-   [Backups](#backups)
-   [Restore a Backup](#restore-a-backup)
-   [Security and GitHub](#security-and-github)
-   [Troubleshooting](#troubleshooting)
-   [Limitations](#limitations)
-   [Roadmap](#roadmap)
-   [License](#license)

## Project Goals

🎯 Tangerine focuses on privacy, control, and keeping the setup
manageable.

-   Run a private messaging server with full control over its
    infrastructure.
-   Use the Matrix protocol for communication.
-   Keep public registration disabled.
-   Avoid relying on public Matrix federation for private communication.
-   Keep resource usage and maintenance requirements low.
-   Explore hosting the server on an Iranian VPS to improve availability
    during international internet disruptions.

## Architecture

🧩 The following components make up the current deployment:

  Component              Technology
  ---------------------- -----------------
  Operating system       Arch Linux
  Messaging protocol     Matrix
  Homeserver             Synapse
  Containerization       Docker
  Container management   Docker Compose
  Database               SQLite
  Private networking     Tailscale
  HTTPS access           Tailscale Serve
  Android client         Element X

### Connection Flow

``` text
Element X (Android)
        |
        v
   Tailscale network
        |
        v
   Arch Linux host
        |
        v
  Tailscale Serve (HTTPS)
        |
        v
    Docker container
        |
        v
   Matrix Synapse
        |
        v
       SQLite
```

## Prerequisites

-   Arch Linux
-   Administrative access to the host
-   Internet access to download container images
-   Docker and Docker Compose
-   A Tailscale account for private device connectivity
-   A Matrix client such as Element X

## Project Structure

``` text
matrix-server/
├── compose.yaml
├── README.md
├── .gitignore
└── synapse-data/
    ├── homeserver.yaml
    └── ...
```

The `synapse-data/` directory contains Synapse configuration, database
files, cryptographic keys, and other persistent server data.

Its exact contents depend on the Synapse version and server activity.

**Never commit `synapse-data/` to a public Git repository.**

The directory shown above is an example. Replace `~/matrix-server` in
the commands below with the actual location of your project if it is
stored elsewhere.

## Install Docker

Update the package database and install Docker and Docker Compose:

``` bash
sudo pacman -Syu docker docker-compose
```

Enable and start Docker:

``` bash
sudo systemctl enable --now docker
```

Check the service status:

``` bash
systemctl status docker
```

Check the installed versions:

``` bash
docker --version
docker compose version
```

Test Docker:

``` bash
sudo docker run --rm hello-world
```

This guide uses `sudo` to avoid adding the regular user to the `docker`
group. Membership in that group effectively grants extensive privileges
over the host system.

## Generate Synapse Configuration

Navigate to your project directory:

``` bash
mkdir -p ~/matrix-server/synapse-data
cd ~/matrix-server
```

The configured Matrix server name for this setup is:

``` text
matrix.tangerine.home.arpa
```

The Matrix server name is not necessarily the same as the HTTPS hostname
used by clients. Choose the server name carefully when creating a new
deployment, because changing it after accounts have been created is not
straightforward.

Generate the initial Synapse configuration:

``` bash
sudo docker run -it --rm \
  -v "$PWD/synapse-data:/data" \
  -e SYNAPSE_SERVER_NAME=matrix.tangerine.home.arpa \
  -e SYNAPSE_REPORT_STATS=no \
  ghcr.io/element-hq/synapse:latest generate
```

This command creates the initial Synapse configuration files in
`synapse-data/`.

**Important:** This step is intended for a fresh installation. If your
existing server already has configuration and database files, do not
regenerate them. Back up your existing data before making changes.

## Configure Docker Compose

Create a `compose.yaml` file in the project root:

``` yaml
services:
  synapse:
    image: ghcr.io/element-hq/synapse:latest
    container_name: matrix-synapse
    restart: unless-stopped
    volumes:
      - ./synapse-data:/data
    ports:
      - "127.0.0.1:8008:8008"
```

### Configuration Explained

-   `image`: Uses the Synapse container image published by Element.
-   `container_name`: Assigns a predictable container name.
-   `restart`: Restarts the container after Docker starts, unless it was
    explicitly stopped.
-   `volumes`: Stores persistent Synapse data outside the container's
    writable layer.
-   `ports`: Exposes port 8008 on the local loopback interface only,
    rather than directly on the host's public network interfaces.

For a stable deployment, consider pinning a specific Synapse image
version after testing it instead of using the mutable `latest` tag.

## Start the Server

Validate the Compose configuration:

``` bash
sudo docker compose config
```

Start Synapse in the background:

``` bash
sudo docker compose up -d
```

Check the container status:

``` bash
sudo docker compose ps
```

Follow the logs:

``` bash
sudo docker compose logs -f synapse
```

Press `Ctrl+C` to stop following the logs. This does not, by itself,
stop the container.

## Manage User Accounts

Public registration should remain disabled.

Open `synapse-data/homeserver.yaml` and verify that the following
setting is present:

``` yaml
enable_registration: false
```

Create a user through Synapse's registration utility:

``` bash
sudo docker exec -it matrix-synapse \
  register_new_matrix_user \
  -c /data/homeserver.yaml \
  http://localhost:8008
```

Follow the interactive prompts to enter the required account details.
Avoid putting passwords directly into shell commands or saved scripts.

To create an administrator account, add the `-a` option:

``` bash
sudo docker exec -it matrix-synapse \
  register_new_matrix_user \
  -c /data/homeserver.yaml \
  -a \
  http://localhost:8008
```

Omit `-a` when creating a regular user.

If account creation fails, check the container status and Synapse logs.
The registration utility requires a running Synapse server.

## Configure Tailscale

Tailscale should be installed and connected on the Arch Linux host.

Check the current Tailnet status:

``` bash
tailscale status
```

Display the host's Tailscale IPv4 address:

``` bash
tailscale ip -4
```

Enable Tailscale at boot:

``` bash
sudo systemctl enable --now tailscaled
```

Check the service:

``` bash
systemctl status tailscaled
```

### Enable Tailscale Serve

To make the local Synapse service available over HTTPS within the
Tailnet:

``` bash
sudo tailscale serve --bg 8008
```

Inspect the Serve configuration:

``` bash
tailscale serve status
```

Keep the HTTPS hostname reported by Tailscale. Use a placeholder in
public documentation instead of publishing your actual hostname:

``` text
https://YOUR-TAILSCALE-HOSTNAME/
```

The address works only while the service is configured, the relevant
devices are authorized, and network connectivity is available.

If you need to remove the HTTPS Serve configuration, inspect
`tailscale serve status` first and use the appropriate command for your
installed Tailscale version.

**Do not use Tailscale Funnel to expose this private server publicly.**

## Connect a Matrix Client

After installing Element X on your device:

1.  Select the manual sign-in option.
2.  If the app defaults to a public server, look for the option to
    change the account provider or homeserver.
3.  Enter the private server's HTTPS address if the client supports it.
4.  Sign in using an existing Matrix account.

A Matrix user ID consists of a username and a server name. For example:

``` text
@USERNAME:matrix.tangerine.home.arpa
```

The HTTPS hostname does not necessarily match the server name in the
Matrix user ID.

Some clients may require server discovery through
`/.well-known/matrix/client` for a smooth sign-in experience.
Compatibility with the chosen client should be tested rather than
assumed.

Successful API access does not, by itself, confirm that Element X
sign-in or messaging is fully configured.

## Health Checks

### Check the Local API

Run:

``` bash
curl -i http://127.0.0.1:8008/_matrix/client/versions
```

A successful response should return HTTP `200` and JSON containing a
`versions` field.

### Check the HTTPS Endpoint

Run:

``` bash
curl -i https://YOUR-TAILSCALE-HOSTNAME/_matrix/client/versions
```

Replace `YOUR-TAILSCALE-HOSTNAME` with the actual Tailscale hostname.

### Inspect Synapse Logs

``` bash
sudo docker compose logs --tail=100 synapse
```

### Monitor Resource Usage

``` bash
sudo docker stats matrix-synapse
```

## Service Management

Start the service:

``` bash
sudo docker compose up -d
```

Stop the Compose services:

``` bash
sudo docker compose down
```

Restart Synapse:

``` bash
sudo docker compose restart synapse
```

Download updated images:

``` bash
sudo docker compose pull
```

After reviewing compatibility and release notes, apply the update:

``` bash
sudo docker compose up -d
```

Back up persistent data before upgrading Synapse, and review any
version-specific migration requirements.

## Backups

Synapse data includes private configuration, database contents,
cryptographic keys, and user information.

For a straightforward SQLite backup, stop the server first:

``` bash
cd ~/matrix-server
sudo docker compose down
```

Create an archive of the data directory:

``` bash
sudo tar -czf "$HOME/tangerine-backup-$(date +%F-%H%M).tar.gz" synapse-data
```

Start the server again:

``` bash
sudo docker compose up -d
```

Store the backup in a secure location. Do not commit it to Git or upload
it to a public location.

For a server with active users, consider a tested backup procedure that
accounts for database consistency, configuration, and recovery
requirements.

## Restore a Backup

**Warning:** Restoring a backup over existing data can destroy newer
messages and configuration. Preserve a separate copy of the current data
before proceeding.

Stop the service:

``` bash
sudo docker compose down
```

Inspect the archive contents before extracting it:

``` bash
tar -tzf ~/tangerine-backup-YYYY-MM-DD-HHMM.tar.gz
```

Replace the example filename with the actual backup filename. Confirm
the archive's directory structure to avoid extracting files into the
wrong location.

Preserve the current data separately, then restore the backup using the
correct destination for the archive's structure. Do not overwrite the
existing data until you have verified both the archive and the
destination.

After restoration, verify file permissions, configuration, and
compatibility with the Synapse version. Start the service:

``` bash
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 synapse
```

Finally, test the local API endpoint.

## Security and GitHub

🔐 Keep private configuration, credentials, and server data out of
version control.

### Recommended `.gitignore`

Create a `.gitignore` file in the project root:

``` gitignore
# Synapse persistent data and private configuration
synapse-data/

# Environment files
.env
.env.*
!.env.example

# Database files
*.db
*.db-shm
*.db-wal

# Private keys and certificates
*.key
*.pem

# Logs and backups
*.log
*.bak
*.backup
*.tar.gz
```

These rules prevent matching untracked files from being added
accidentally. They do not automatically remove files that Git already
tracks.

### Inspect Git Status

Check the working tree:

``` bash
git status --short
```

List tracked files:

``` bash
git ls-files
```

Check whether the Synapse configuration is ignored:

``` bash
git check-ignore -v synapse-data/homeserver.yaml
```

If a sensitive file is already tracked, adding it to `.gitignore` is not
sufficient to remove it from the repository.

### Initialize the Repository

From the directory containing the files intended for publication:

``` bash
git init
git add README.md compose.yaml .gitignore
git status
```

Review the staged file list before committing. If any private data
appears, stop and correct the staging area first.

Commit the documentation and configuration template:

``` bash
git commit -m "Add Tangerine Matrix server documentation"
```

Configure the GitHub remote and push only after reviewing the repository
visibility, access permissions, and staged files.

### Never Publish These Items

-   User passwords
-   `registration_shared_secret`
-   `macaroon_secret_key`
-   `form_secret`
-   The server signing key
-   Access tokens and API keys
-   SQLite databases and user data
-   The entire `synapse-data/` directory
-   Server backups
-   Tailscale authentication credentials
-   Unnecessary personal information or private identifiers

If a secret has already been pushed, deleting it from the current
version of a file is not enough. Review the Git history and rotate any
exposed credentials or keys as appropriate.

## Troubleshooting

### The Container Does Not Start

``` bash
sudo docker compose ps
sudo docker compose logs --tail=100 synapse
```

### Docker Does Not Start at Boot

``` bash
systemctl is-enabled docker
systemctl status docker
```

If necessary:

``` bash
sudo systemctl enable --now docker
```

### The Local API Does Not Respond

``` bash
curl -v http://127.0.0.1:8008/_matrix/client/versions
sudo docker compose logs --tail=100 synapse
```

### Tailscale Is Disconnected

``` bash
tailscale status
tailscale ip -4
systemctl status tailscaled
```

### The HTTPS Endpoint Does Not Respond

``` bash
tailscale serve status
curl -v https://YOUR-TAILSCALE-HOSTNAME/_matrix/client/versions
```

Also verify that the client device is connected to the Tailnet and is
authorized to reach the host.

### Element X Cannot Sign In

-   Verify the HTTPS hostname.
-   Confirm that the server API responds successfully.
-   Make sure the Matrix server name matches the account's server
    identifier.
-   Check whether `/.well-known/matrix/client` discovery is required.
-   Record the exact client error for troubleshooting, but never publish
    passwords or access tokens.

## Limitations

-   The server hosted on a laptop is unavailable when the laptop is
    powered off.
-   Tailscale alone cannot guarantee connectivity during a complete
    international internet shutdown.
-   Hosting the server in Iran may help only if the internal network
    paths between users and the server remain available.
-   Full compatibility with Element X and the private server hostname
    still needs to be verified.
-   SQLite is suitable for a small deployment, but future requirements
    may justify migrating to PostgreSQL.
-   Backups and restoration should be tested before the server is relied
    upon for important communication.

## Roadmap

🛣️ Planned work and current verification status:

-   [x] Deploy Synapse using Docker Compose
-   [x] Create an initial Matrix account
-   [x] Verify the local Synapse API
-   [x] Configure Tailscale Serve
-   [x] Receive an API response over HTTPS
-   [ ] Complete and verify Element X sign-in
-   [ ] Test sending and receiving messages
-   [ ] Test backup restoration
-   [ ] Review and improve security documentation
-   [ ] Evaluate an Iranian VPS
-   [ ] Migrate to a VPS if feasible
-   [ ] Test connectivity during international internet disruptions

## License

📜 Tangerine is licensed under the **MIT License**. See the
[`LICENSE`](LICENSE) file for the full license text.

The MIT License allows use, copying, modification, distribution,
sublicensing, and sale of copies of the software, provided that the
copyright and license notice are included.

## AI Disclosure

🤖 This README was written with the assistance of AI.
