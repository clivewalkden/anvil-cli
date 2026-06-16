# Anvil

![GitHub Release](https://img.shields.io/github/v/release/clivewalkden/anvil-cli)

A CLI tool to manage Magento Docker environments — wraps `docker compose`, runs Magento/WordPress tooling inside containers, and syncs configurable Docker template files from a remote repository.

## Features

- Syncs Docker template files from `clivewalkden/magento-docker-templates`
- Full container lifecycle management (start, stop, restart, build, logs, etc.)
- Magento CLI wrappers (cache management, n98-magerun, ece-patches)
- Frontend build tooling (composer, npm, node, grunt, Hyva/Tailwind)
- Database download, export, and import for Magento and WordPress
- Media sync via rsync from remote servers
- File copy between host and container via `docker cp`
- Environment setup orchestration (domains, SSL, RabbitMQ, Elasticsearch, WordPress)
- WordPress CLI passthrough
- Dry-run mode for template sync

## Requirements

- Go 1.23+
- Docker with the Compose plugin
- A `docker-sync.yaml` config file in your project root (see [Configuration](#configuration))

## Installation

### Homebrew (macOS/Linux)

```sh
brew install clivewalkden/taps/anvil
```

### Build from source

```sh
git clone git@github.com:clivewalkden/go-sozo-magento-docker-manager.git
cd go-sozo-magento-docker-manager
make build
```

The compiled binary is written to `bin/`.

## Usage

```sh
anvil [--verbose|-v] [--version|-V] <command>
```

Use `-v`, `-vv`, or `-vvv` to increase log verbosity. All commands inherit these flags.

---

## Commands

### `anvil init`

Interactively creates a `docker-sync.yaml` configuration file in the current directory. Prompts for required fields (PHP, MySQL, Redis versions), server provider, and optional feature flags, writing a commented config file ready for `anvil sync`.

Selecting **CoreFinity** as the server provider additionally prompts for `cf_remote_bin_dir` (the remote bin directory used by CoreFinity backup scripts). This value is written to `docker-sync.yaml` and applied to `.docker/config.env` automatically during `anvil sync`.

```sh
anvil init
```

---

### `anvil sync`

Syncs Docker template files from the remote template repository into the current project. Validates `docker-sync.yaml` before running — missing required fields are reported upfront. Skips files listed in `.anvilignore`. Use `--force` to overwrite add-only files.

```sh
anvil sync
anvil sync --force
anvil sync --dry-run
```

---

### `anvil docker` — Container lifecycle

```sh
anvil docker start                   # docker compose up -d
anvil docker start --build           # Rebuild images before starting
anvil docker start --remove-orphans  # Remove orphaned containers on start
anvil docker stop                    # docker compose stop
anvil docker stop --optimise         # Optimise (clear cache, DB, logs) then stop
anvil docker restart                 # Stop then start
anvil docker build                   # Rebuild images without cache
anvil docker status                  # docker compose ps
anvil docker check                   # docker compose config (validate)
anvil docker cleanup                 # docker compose down --volumes
anvil docker optimise                # Clear image cache, optimise DB, truncate logs
anvil docker logs                    # Follow all service logs
anvil docker logs <service>          # Follow a specific service's logs
```

#### `anvil docker cli` — Exec into the container

Runs a command inside the `fpm` container. All arguments after any flags are passed through to the container.

```sh
anvil docker cli bash                           # Open a bash shell
anvil docker cli bin/magento cache:status       # Run any command
anvil docker cli --user root bash               # Run as root
anvil docker cli --no-tty ls /app               # No pseudo-TTY (-T)
anvil docker cli --user root --no-tty chmod +x /app/bin/magento
anvil docker cli --container db mysql -u root   # Target a different container
```

| Flag | Default | Description |
| --- | --- | --- |
| `--user` / `-u` | `www-data` | Container user to run as |
| `--container` / `-c` | `fpm` | Target container |
| `--no-tty` / `-T` | `false` | Disable pseudo-TTY (for scripting) |

---

### `anvil magento` — Magento CLI

```sh
anvil magento run <args>             # bin/magento <args>
anvil magento n98 <args>             # n98-magerun <args>
anvil magento ece-patches <args>     # ece-patches <args>
```

#### Cache management

```sh
anvil magento cache clear            # cache:clear
anvil magento cache clean            # cache:clean
anvil magento cache flush            # cache:flush
anvil magento cache enable           # cache:enable
anvil magento cache disable          # cache:disable
anvil magento cache status           # cache:status
```

---

### `anvil build` — Frontend build tools

All commands run inside the `fpm` container as `www-data`.

```sh
anvil build composer <args>          # composer <args>
anvil build npm <args>               # npm <args>
anvil build node <args>              # node <args>
anvil build grunt <args>             # Deletes pub/static, then npx grunt <args>
anvil build compile-hyva <args>      # npm run <args> inside TAILWIND_DIR
anvil build setup-hyva               # rm node_modules + npm install in TAILWIND_DIR
anvil build setup-frontend           # Full grunt/npm frontend setup
```

`setup-frontend` queries the active Magento theme from the database and runs the full npm install + grunt compilation pipeline.

`compile-hyva` and `setup-hyva` require `TAILWIND_DIR` to be set in `.docker/config.env`.

---

### `anvil db` — Database management

```sh
# Download from remote server (SSH + SCP)
anvil db download                    # Interactive: prompt for Magento + WordPress
anvil db download --magento          # Magento only
anvil db download --wordpress        # WordPress only
anvil db download -y                 # Skip prompts

# Export from container
anvil db export                      # Interactive: prompt for Magento + WordPress
anvil db export --magento [file]     # Magento only (default: m2.sql)
anvil db export --wordpress [file]   # WordPress only (default: wp.sql)
anvil db export-table <table>        # Export a single table

# Import into container
anvil db import                      # Interactive: prompt for Magento + WordPress
anvil db import --magento [file]     # Magento only (default: /app/dev/m2.sql)
anvil db import --wordpress [file]   # WordPress only (default: /app/dev/wp.sql)
anvil db import -y                   # Skip prompts
```

**Download** uses these env vars from `.docker/config.env`:

| Var | Purpose |
| --- | --- |
| `SSH_HOST` | Remote SSH host |
| `SERVER_PROVIDER` | Controls backup strategy (`CoreFinity`, `Sonassi`, or `GCloud`) |
| `CF_REMOTE_BIN_DIR` | CoreFinity only — remote bin directory containing backup scripts |
| `REMOTE_MAGENTO_DIR` | Remote Magento root (Sonassi / GCloud only) |
| `REMOTE_PHP_BIN` | Remote PHP binary path (Sonassi / GCloud only) |
| `REMOTE_N98_MAGERUN_BIN` | Remote n98-magerun binary path (all providers, Magento only) |
| `REMOTE_WP_CLI_BIN` | Remote WP CLI binary path (Sonassi / GCloud, WordPress only) |
| `REMOTE_DB_BACKUP_DIR` | Remote directory for DB dumps |
| `APP_SCHEMA` | URL scheme for domain replacement (default: `https`) |
| `LOCAL_DOMAINS` | Comma-separated local domains |
| `REMOTE_DOMAINS` | Comma-separated live domains (replaced in download) |

**Magento import** automatically runs `setup:upgrade`, `indexer:reindex`, and `cache:flush` after importing.

**WordPress import** performs domain search-replace via WP CLI using `LOCAL_DOMAINS` / `REMOTE_DOMAINS`.

WordPress commands require `enable_wordpress: true` in `docker-sync.yaml`.

---

### `anvil media` — Media file management

```sh
anvil media import                   # Interactive: prompt for Magento + WordPress
anvil media import --magento         # rsync Magento pub/media from remote
anvil media import --wordpress       # rsync WordPress uploads from remote
anvil media import -y                # Skip prompts
```

Uses these env vars from `.docker/config.env`:

| Var | Purpose |
| --- | --- |
| `SSH_HOST` | Remote SSH host |
| `SERVER_PORT` | SSH port (omit to use the default port 22) |
| `REMOTE_SHARED_DIR` | Remote shared media directory |
| `REMOTE_WP_DIR` | WordPress directory name under `REMOTE_SHARED_DIR` (default: `wp`) |

WordPress requires `enable_wordpress: true` in `docker-sync.yaml`.

---

### `anvil files` — File copy between host and container

```sh
anvil files pull <path>              # Copy /app/<path> from container → magento/<path>
anvil files pull --all               # Copy entire /app from container → magento/
anvil files push <path>              # Copy magento/<path> from host → /app/<path>
anvil files push --all               # Copy entire magento/ from host → /app/
anvil files push <path> -y           # Skip permission-fix prompt after push
anvil files diff <path>              # Show diff between host and container versions of a file
```

`files diff` compares `magento/<path>` on the host against `/app/<path>` in the `fpm` container and prints a unified diff. If differences are found you are prompted to push (host → container), pull (container → host), or skip.

---

### `anvil setup` — Environment setup

```sh
anvil setup run                      # Full setup orchestration (see below)
anvil setup run -y                   # Skip all interactive prompts

anvil setup domain                   # Add domains to /etc/hosts + set Magento base URLs
anvil setup ssl <domain>             # Generate SSL cert and fetch from tls container
anvil setup ssl-ca                   # Install SOZO CA cert on macOS or Linux

anvil setup ssh                      # Copy SSH key into fpm container
anvil setup rabbitmq                 # Create magento RabbitMQ user, remove guest

anvil setup elasticsearch            # Configure ES8 (indices.id_field_data.enabled)

anvil setup wordpress                # Deploy FishPig theme + mu-plugins
anvil setup wordpress-local          # Symlink wp-config, child theme, mu-plugins
```

#### `anvil setup run` — Full orchestration

Runs the complete setup sequence in order:

1. `docker start --build --remove-orphans`
2. Fix `magento/` file permissions
3. Copy SSH key into container
4. `composer install -n`
5. `db download` (interactive)
6. Wait for SQL dump files to be present
7. `db import` (interactive)
8. Press Enter prompt before Magento setup commands
9. `setup:upgrade` → `setup:static-content:deploy` → `indexer:reindex` → `cache:flush`
10. `setup wordpress` _(if `EnableWordpress: true`)_
11. `build setup-hyva` + `build compile-hyva build` _(if `TAILWIND_DIR` is set)_, or `build setup-frontend`
12. `media import` (interactive)
13. `setup domain`
14. `setup rabbitmq`
15. `setup elasticsearch` _(if `ElasticSearchVersion >= 8.0.0`)_

---

### `anvil wordpress` — WP CLI passthrough

```sh
anvil wordpress <args>               # wp --path=/app/wp --color <args>
```

Requires `enable_wordpress: true` in `docker-sync.yaml`. All arguments are passed directly to WP CLI inside the `fpm` container.

---

### `anvil system` — System administration

```sh
anvil system permissions             # chmod 2666 (files) + 2777 (dirs) in magento/
anvil system elasticsearch <args>    # docker compose exec elasticsearch <args>
```

---

### `anvil completion` — Shell completion

Generates a shell completion script. Once installed, pressing Tab completes subcommands and flags.

```sh
anvil completion bash
anvil completion zsh
anvil completion fish
anvil completion powershell
```

Run `anvil completion --help` for shell-specific installation instructions.

---

## Configuration

Create `docker-sync.yaml` in your project root using `anvil init` or by copying `example.config.yaml`. The three required fields are validated before `anvil sync` runs.

```yaml
# --- Required ---
php_version: "8.2"
mysql_version: "8.0"
redis_version: "7.0"

# --- Server ---
server_provider: "Sonassi"   # CoreFinity | Sonassi | GCloud

# CoreFinity only — remote bin directory for backup scripts
# cf_remote_bin_dir: "/home/corefinity/bin"

# --- Optional services ---
elasticsearch_version: "8.7"
rabbitmq_version: "3.11"

# --- Feature flags ---
enable_kibana: false
kibana_version: "8.11.4"

enable_hyva: false
hyva_theme_name: "clientname"

enable_varnish: false
varnish_version: "6.0"

enable_wordpress: false
enable_xdebug: false

# --- Binary paths (inside the fpm container) ---
php_binary: "/usr/local/bin/php"
n98_binary: "/usr/local/bin/n98-magerun2"
wp_cli_binary: "/usr/local/bin/wp"
```

### Key config flags

| Flag | Effect |
| --- | --- |
| `server_provider` | Controls remote DB backup strategy — `CoreFinity` uses remote backup scripts; `Sonassi` / `GCloud` use n98-magerun2 directly |
| `enable_wordpress: true` | Enables `anvil wordpress`, `anvil db *--wordpress`, `anvil media import --wordpress`, `anvil setup wordpress*` |
| `elasticsearch_version: "8.x"` | `anvil setup run` calls `anvil setup elasticsearch` when version >= 8.0.0 |
| `hyva_theme_name` | Used by template rendering for Hyva/Tailwind setup |

### Environment files

Anvil loads environment variables at runtime from:

- `.docker/config.env` — required; project-level variables
- `.docker/local.env` — optional; local overrides (gitignored)

`USER_ID` and `GROUP_ID` are set automatically from the current user.

---

## License

MIT
