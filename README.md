# Minecraft Server Setup

This repository contains a containerized Minecraft server setup with:

- `minecraft-server` using `itzg/minecraft-server`
- `minecraft-data-backup` using `itzg/mc-backup`
- `ddns-updater` using `qmcgaw/ddns-updater` for No-IP updates

## Project layout

- `docker-compose.yml` - portable base Compose file
- `docker-compose.selinux.yml` - SELinux override
- `.env` - local environment variables and secrets
- `bootstrap.sh` - initial setup helper for restoring or initializing data
- `minecraft-data/` - local Minecraft server data directory
- `config/` - local rclone config directory containing `rclone/rclone.conf`

## Requirements

You will need:

- Docker Compose or Podman Compose
- a `.env` file in the project root
- a valid `rclone.conf` under `./config/rclone/` if you want remote backups/restores
- a populated `minecraft-data/` directory, or use `bootstrap.sh` to create/restore it

## Required local configuration files

This setup relies on two local configuration files that you provide on the host:

1. `.env`
2. `./config/rclone/rclone.conf`

### 1. `.env`

Location:

- project root as `./.env`

Purpose:

- stores environment variables and secrets used by the Minecraft server and No-IP updater

Required values:

- `RCON_PASSWORD`
- `NOIP_USERNAME`
- `NOIP_PASSWORD`
- `NOIP_HOSTNAMES`

Optional values:

- `TZ`

Example:

```/dev/null/.env#L1-5
RCON_PASSWORD=your-rcon-password
NOIP_USERNAME=your-noip-username
NOIP_PASSWORD=your-noip-password
NOIP_HOSTNAMES=yourhost.ddns.net
TZ=America/New_York
```

### 2. `rclone.conf`

Location:

- `./config/rclone/rclone.conf`

Purpose:

- stores the rclone remote configuration used by the backup container and bootstrap restore flow to access Google Drive

What it must contain:

- an rclone remote section matching the configured remote name in Compose
- for the current setup, that means a remote named `gdrive`

Example remote section:

```/dev/null/rclone.conf#L1-4
[gdrive]
type = drive
scope = drive
token = {"access_token":"...","token_type":"Bearer","refresh_token":"...","expiry":"..."}
```

If this file is missing, or if it does not contain a `[gdrive]` section, Google Drive backups and remote restores will fail.



## Environment variables

Your `.env` file should define at least:

- `RCON_PASSWORD`
- `NOIP_USERNAME`
- `NOIP_PASSWORD`
- `NOIP_HOSTNAMES`

Optional:

- `TZ`

The Compose configuration validates the required variables above at startup. If any of them are missing, Compose will fail fast with a clear error instead of starting a broken container configuration.

Example:

```/dev/null/.env#L1-5
RCON_PASSWORD=your-rcon-password
NOIP_USERNAME=your-noip-username
NOIP_PASSWORD=your-noip-password
NOIP_HOSTNAMES=yourhost.ddns.net
TZ=America/New_York
```

## Running the stack

### Most systems

For macOS and non-SELinux Linux systems, use the base Compose file only:

```/dev/null/commands.sh#L1-1
docker compose up -d
```

Or with Podman:

```/dev/null/commands.sh#L1-1
podman compose up -d
```

On macOS with Docker Desktop:

- use `docker-compose.yml` only
- do not use `docker-compose.selinux.yml`
- make sure Docker Desktop is allowed to access the project directory
- make sure both required local config files are present:
  - `./.env`
  - `./config/rclone/rclone.conf`

### SELinux-enabled systems

Use the base Compose file together with the SELinux override:

```/dev/null/commands.sh#L1-1
podman compose -f docker-compose.yml -f docker-compose.selinux.yml up -d
```

This applies SELinux-friendly shared `:z` bind mount options without making the main Compose file less portable.

## SELinux override behavior

The file `docker-compose.selinux.yml` is a small override file. It only replaces the bind mount definitions that need SELinux relabeling.

It uses shared `:z` labels rather than private `:Z` labels because both `minecraft-server` and `minecraft-data-backup` need to access the same `./minecraft-data` bind mount. Shared labels are the better fit for this multi-container SELinux setup.

The override is merged on top of `docker-compose.yml` when both files are passed on the command line.

## Initial setup

If `minecraft-data/` is empty, you can use the bootstrap helper:

```/dev/null/commands.sh#L1-2
chmod +x bootstrap.sh
./bootstrap.sh
```

If `minecraft-data/` already contains files, the script exits early by default. To force the interactive prompts anyway, run:

```/dev/null/commands.sh#L1-1
FORCE_PROMPT=1 ./bootstrap.sh
```

On SELinux-enabled systems, the bootstrap restore path applies SELinux-aware bind mount labeling automatically when using Podman. This helps the temporary rclone container read `./config/rclone.conf` and write into `./minecraft-data` during restore.

The script can:

1. copy an existing local Minecraft data directory into `./minecraft-data`
2. restore Minecraft data from an rclone remote into `./minecraft-data` using the project's `./config/rclone/rclone.conf`
3. leave `./minecraft-data` empty and let the server initialize it on first startup

For the restore option to work, both required local config files need to be in place first:

- `./.env` for the stack configuration
- `./config/rclone/rclone.conf` for the rclone remote definition

When restoring from an rclone remote, enter a source path that matches a remote configured in `./config/rclone/rclone.conf`.

Examples:

- `gdrive:Minecraft Server Data Backups`
- `gdrive:"Minecraft Server Data Backups"`

If you are unsure which remote names are available, open `./config/rclone/rclone.conf` and look for section headers such as:

- `[gdrive]`
- `[some-other-remote]`

## Backups

The backup service uses `itzg/mc-backup` and uploads backups using rclone.

It expects:

- Minecraft server data at `./minecraft-data`
- rclone configuration file at `./config/rclone/rclone.conf`

To configure backups successfully:

- keep your backup destination settings in `docker-compose.yml`
- keep your Google Drive remote definition in `./config/rclone/rclone.conf`
- make sure the remote name in `rclone.conf` matches `RCLONE_REMOTE` in Compose, which is currently `gdrive`

On SELinux-enabled systems, use the SELinux override when running the stack so the shared bind mounts are mounted with the required SELinux labels.

The backup container expects the project rclone configuration to be available at `/config/rclone/rclone.conf` inside the container, which maps from `./config/rclone/rclone.conf` on the host.

### If you copied `minecraft-data` from somewhere else

If you directly copy an existing Minecraft data directory into `./minecraft-data`, the copied files may keep permissions and ownership that do not work well with this container setup.

Symptoms can include:

- the Minecraft server cannot write files such as `logs/latest.log`
- the backup container fails with errors like `tar: .: Cannot stat: Permission denied`
- players get disconnected because the server cannot write required files and stops

On SELinux-enabled systems, the most reliable recovery sequence is:

```/dev/null/commands.sh#L1-5
podman-compose -f docker-compose.yml -f docker-compose.selinux.yml down
sudo chown -R "$USER":"$USER" minecraft-data
sudo find minecraft-data -type d -exec chmod 755 {} \;
sudo find minecraft-data -type f -exec chmod 644 {} \;
podman-compose -f docker-compose.yml -f docker-compose.selinux.yml up -d
```

This fixes the common case where copied files are too restrictive for the server or backup container. The SELinux override also applies shared SELinux labels using `:z`, which is important because both the Minecraft server and backup container need access to the same `./minecraft-data` directory.

Current backup target configuration is driven by the Compose file:
- remote: `gdrive`
- destination directory: `Backups/Minecraft Daily Backups`
- backup on startup: enabled
- backup interval: every 24 hours
- retention: keep the last 15 backups

## RCON access

RCON is enabled for internal container-to-container use, but it is not exposed on the host.

This means:

- `minecraft-data-backup` can still reach the Minecraft server over the internal Compose network
- host-side clients cannot connect directly to `localhost:25575` unless you re-add that port mapping

This keeps RCON less exposed while still allowing the backup container to use it.

## DDNS updates

The `ddns-updater` service uses your No-IP credentials from `.env` and keeps your configured hostname(s) updated.

It uses:

- `NOIP_USERNAME`
- `NOIP_PASSWORD`
- `NOIP_HOSTNAMES`

## Notes

- Do not commit `.env`
- Do not commit `minecraft-data/`
- Do not commit secret rclone credentials under `config/`
- On SELinux-enabled systems, prefer the override command shown above
