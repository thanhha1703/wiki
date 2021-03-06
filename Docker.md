---
title: Docker
---

> #### Updating from an old container
> Since v2.0.7, the Docker container no longer uses the root user by default and therefore old setups may break.
> You can either force the old behaviour by adding the `--user root` flag to `docker run`, or make sure to change your config and data directories to ones accessible by the `parity` user (uid 1000) within the docker container or before adding the directories as volumes to docker.

Build and run Parity Ethereum anywhere with Docker:

- [Pick a container](#pick-a-container)
- [Run container](#run-container)
- [Configure Parity Ethereum](#configure-parity-ethereum)
    - [Open ports](#open-ports)
    - [Pass arguments](#pass-arguments)
    - [Config file](#config-file)
    - [Persistent data directory](#persistent-data-directory)
    - [Run in background](#run-in-background)
- [Parity Ethereum Deploy Scripts](#parity-ethereum-deploy-scripts)
    - [Prepare the Node](#prepare-the-node)
    - [Launching Parity Ethereum](#launching-parity-ethereum)
    - [More Options](#more-options)
- [Further reading](#further-reading)


### Pick a container

Docker containers for Parity Ethereum are available via [Docker Hub](https://hub.docker.com/r/parity/parity/):

```bash
$ docker search parity/parity
NAME                        DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
parity/parity               Parity is Ethcore's initial fully-featured...   0
parity/rust                 Rust stable, beta and nightly for GitLab C...   0                    [OK]
parity/snapcraft            Docker image for build snap application (U...   0                    [OK]
parity/rust-arm             RUST for GitLab CI runner (ARM architecture)    0                    [OK]
```

To get a list of available versions, use `curl` and `jq`:

```bash
$ curl -sS 'https://registry.hub.docker.com/v2/repositories/parity/parity/tags/'  | jq '."results"[]["name"]' | sort
"beta"
"nightly"
"stable"
"v2.0.6"
"v2.0.7"
"v2.0.8"
"v2.1.2"
"v2.1.3"
"v2.1.4"
"v2.2.0-rc6"
```

To get the latest beta release, run:

```bash
$ docker pull parity/parity:v2.1.3
```

### Run container

To run Parity Ethereum with an interactive pseudo-tty shell, run:

```bash
$ docker run -ti parity/parity:v2.1.3
```

### Configure Parity Ethereum

Parity can be configured using either the [CLI options or a config file](Configuring-Parity.md). Should the CLI flags and the config file disagree about a setting, the CLI takes precedence. You can list all CLI options by running:

```bash
$ docker run parity/parity:v2.1.3 --help
```

For Docker specific options, please refer to the [Docker documentation](https://docs.docker.com/engine/), or run `docker --help` or `docker run --help`.

##### Open ports

To publish Parity Ethereum's ports to the host machine, use the `-p` option:

```bash
$ docker run -ti -p 8545:8545 -p 8546:8546 -p 30303:30303 -p 30303:30303/udp parity/parity:v2.1.3 --ui-interface all --jsonrpc-interface all
```

For example, this will expose the HTTP and WebSockets JSONRPC APIs, and the listen port to the host. Now you can send RPC calls from the Docker host computer.

To enable external discovery (for example for PoA sealing nodes), specify the external IP by appending the flag `--nat extip:133.3.3.37`, where `133.3.3.37` is to be replaced by your actual external IP of the host.

##### Pass arguments

To pass further operating options to Parity Ethereum, simply append them to the `docker run` command:

```bash
$ docker run -ti parity/parity:v2.1.3 --no-ui --no-dapps --no-discovery
```

In this case, it disables the Wallet, the DApps Server, and discovery.

##### Config file

For more complex node configurations, a TOML config file can be created and attached to the docker container.

```bash
$ mkdir -p ~/.local/share/io.parity.ethereum/docker/
$ touch ~/.local/share/io.parity.ethereum/docker/config.toml
```

You can use the [Config File Generator](https://paritytech.github.io/parity-config-generator/) to configure your Parity Ethereum node and save it on your host's disk, i.e., at `~/.local/share/io.parity.ethereum/docker/config.toml`. To mount the configuration, use the `docker run -v` option:

```bash
$ docker run -ti -v ~/.local/share/io.parity.ethereum/docker/:/home/parity/.local/share/io.parity.ethereum/ parity/parity:v2.1.3 --config /home/parity/.local/share/io.parity.ethereum/config.toml
```

This will mount `~/.local/share/io.parity.ethereum/docker/` of the host machine at `/home/parity/.local/share/io.parity.ethereum/` inside the docker container. Therefore, the config file will be available via `--config /home/parity/.local/share/io.parity.ethereum/config.toml`.

##### Persistent data directory

In case you need to persist the blockchain files, keys etc., you should run the image with the `--base-path` option and then mount it, e.g.:

```
$ docker run -ti -v ~/.local/share/io.parity.ethereum/docker/:/home/parity/.local/share/io.parity.ethereum/ parity/parity:v2.1.3 --base-path /home/parity/.local/share/io.parity.ethereum/
```

This will expose the whole data dir to the host machine at `~/.local/share/io.parity.ethereum/docker/`.

Windows machines don't support unix permissions, which means you will likely experience errors when mounting a local volume as a non-root user. One workaround for this is to create a volume using:

```
docker volume create --driver=local --opt o=uid=1000 --opt type=tmpfs --opt device=tmpfs paritydb
```
This ensures that the volume has the correct permissions to give the `parity` user access to it.

Your can then mount the volume with:

```
$ docker run -ti -v paritydb:/home/parity/.local/share/io.parity.ethereum/ parity/parity:v2.1.3 --base-path /home/parity/.local/share/io.parity.ethereum/
```

##### Run in background

To run a detached Parity Ethereum instance, use `docker run -d`:

```bash
$ docker run -d parity/parity:v2.1.3
245f312f3f39ad0a518091b1ee4cdc0c1f6d74fb9609395ed3fdcf43acae4b62
```

It will run Parity Ethereum in background. `docker ps` shows the instance:

```bash
$ docker ps
CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                          NAMES
245f312f3f39        parity/parity:v2.1.3   "/parity/parity"    7 seconds ago       Up 6 seconds        8080/tcp, 8180/tcp, 8545/tcp   epic_pike
```

To attach the container, use `docker attach`:

```bash
$ docker attach --sig-proxy=false 245f312f3f39
```

Disabling the signal proxy [allows to detach again](http://stackoverflow.com/a/22894096) with `CTRL`+`C`.

### Parity Ethereum Deploy Scripts

A Parity Ethereum deployment script generator is available at [paritytech/parity-deploy](https://github.com/paritytech/parity-deploy). It uses `docker` and `docker-compose`. On Ubuntu systems these will automatically be installed if not already present on the system.

Currently these scripts supports two types of chains, either [instant sealing](https://wiki.parity.io/Pluggable-Consensus#instant-seal) for development and [authority round](https://wiki.parity.io/Pluggable-Consensus#aura) for proof of authority with multiple validators.

##### Prepare the node

Some examples of using the script are:

- A single node instant seal node, accessible via [Parity UI](https://github.com/Parity-JS/shell/releases):

  ```bash
  $ ./parity-deploy.sh --name testchain --config instantseal
  ```

- A three node proof of authority chain (using the Aura consensus algorithm) with one client accessible via [Parity UI](https://github.com/Parity-JS/shell/releases):

  ```bash
  $ ./parity-deploy.sh --name testchain --config aura --nodes 3
  ```

##### Launching Parity Ethereum

Once the configuration is created you just need to run the docker-compose command to launch the machine or machines. This can be done via:

```bash
$ docker-compose up -d
```

You will then be able to see the logs by running:

```bash
$ docker-compose logs -f
```

In these logs you should see a token being generated to login to Parity Ethereum. Alternatively you can run the command:

```bash
$ docker-compose logs | grep token
```

Once you are logged into the web interface if you go to _Add Accounts_, then select the option recovery phrase and enter the account recovery phrase as password. You now have an account with lots of ether to send around.

##### More options

You can also include extra nodes (e.g. [ethstats monitoring](https://github.com/cubedro/eth-net-intelligence-api)) by including the docker-compose configuration in `include/docker-compose.yml`. To add Ethstats monitoring you would need to include this in the file:

```
  monitor:
    image: buythewhale/ethstats_monitor
    volumes:
      - ./monitor/app.json:/home/ethnetintel/eth-net-intelligence-api/app.json:ro
  dashboard:
    image: buythewhale/ethstats
    volumes:
      - ./dashboard/ws_secret.json:/eth-netstats/ws_secret.json:ro
    ports:
      - 3001:3000
```

### Further reading

- [Running a Parity Ethereum node in Docker and connect safely](https://medium.com/@preitsma/setting-up-a-parity-ethereum-node-in-docker-and-connect-safely-f881faa17686).
- [How to quickly upgrade an Ethereum Parity node using Docker](https://medium.com/decentralized-capital/how-to-quickly-upgrade-an-ethereum-parity-node-using-docker-e170fa2a2045).
- [Docker compose network with 3 authorities and netstats dashboard](https://github.com/dstarcev/parity-poa-playground).
- [Docker compose for running integration tests](https://github.com/illya13/parity-poa).
- [Building Parity using Docker](Setup.md#Building-using-Docker).
- [Docker build for ARM64](https://github.com/paritytech/parity-snappy/wiki/Docker-build-for-ARM-ARM64).
- [Parity Ethereum on Docker Hub](https://hub.docker.com/r/parity/parity/).
- [View the Dockerfile here](https://github.com/paritytech/parity-ethereum/blob/master/docker/ubuntu/Dockerfile).
