# Anvil

A CLI tool to manage Magento Docker environments using configurable templates and service versions.

## Features

- Syncs Docker template files for Magento projects
- Supports dry-run mode to preview changes
- Configurable versions for PHP, Redis, MySQL, RabbitMQ, and Elasticsearch

## Requirements

- Go 1.23+
- Docker

## Installation

Clone the repository and build the binary:

```sh
git clone git@github.com:yourorg/go-sozo-magento-docker-manager.git
cd go-sozo-magento-docker-manager
go build -o anvil
```

## Usage

Run the tool with the following command:

```sh
./anvil
```

To preview changes without applying them, use the `--dry-run` flag:

```sh
./anvil --dry-run
```

### Commands

- `check`: Checks the current Docker environment and configuration.
- `cleanup`: Cleans up the Docker environment by removing project containers, images, and volumes.
- `optimise`: Optimizes the Docker environment by cleaning up unused resources.
- `restart`: Restarts the Docker containers using the current configuration.
- `start`: Starts the Docker containers using the synced templates.
- `status`: Displays the status of the Docker containers.
- `stop`: Stops the running Docker containers. Use the `--optimise` flag to clean up unused resources.
- `sync`: Syncs the Docker template files based on the configuration. Use the `--force` flag to overwrite existing files.


## Configuration

Edit the `example.config.yaml` file to set your desired versions for each service. The file should look like this:

```yaml
docker_template_version: "20250613"
elasticsearch_version: "8.7"
enable_kibana: false
enable_hyva: false
enable_varnish: false
enable_wordpress: true
hyva_theme_name: "clientname"
kibana_version: "8.11.4"
mysql_version: "8.0"
php_version: "8.2"
rabbitmq_version: "3.11"
redis_version: "7.0"
varnish_version: "6.0"
php_binary: "/usr/local/bin/php"
n98_binary: "/usr/local/bin/n98-magerun2"
wp_cli_binary: "/usr/local/bin/wp"
```

### License

TBC
