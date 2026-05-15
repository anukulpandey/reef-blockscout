# Docker-compose configuration

Runs Blockscout locally in Docker containers with [docker-compose](https://github.com/docker/compose).

## Prerequisites

- Docker v20.10+
- Docker-compose 2.x.x+
- Running Ethereum JSON RPC client

## Building Docker containers from source

**Note**: in all below examples, you can use `docker compose` instead of `docker-compose`, if compose v2 plugin is installed in Docker.

```bash
cd ./docker-compose
docker-compose up --build
```

**Note**: if you don't need to make backend customizations, you can run `docker-compose up` in order to launch from pre-build backend Docker image. This will be much faster.

This command uses `docker-compose.yml` by-default, which builds the backend of the explorer into the Docker image and runs 9 Docker containers:

- Postgres 14.x database, which will be available at port 7432 on the host machine.
- Redis database of the latest version.
- Blockscout backend with api at /api path.
- Nginx proxy to bind backend, frontend and microservices.
- Blockscout explorer at http://localhost.

and 5 containers for microservices (written in Rust):

- [Stats](https://github.com/blockscout/blockscout-rs/tree/main/stats) service with a separate Postgres 14 DB.
- [Sol2UML visualizer](https://github.com/blockscout/blockscout-rs/tree/main/visualizer) service.
- [Sig-provider](https://github.com/blockscout/blockscout-rs/tree/main/sig-provider) service.
- [User-ops-indexer](https://github.com/blockscout/blockscout-rs/tree/main/user-ops-indexer) service.

**Note for Linux users**: Linux users need to run the local node on http://0.0.0.0/ rather than http://127.0.0.1/

## Configs for different Ethereum clients

The repo contains built-in configs for different JSON RPC clients without need to build the image.

| __JSON RPC Client__    | __Docker compose launch command__ |
| -------- | ------- |
| Erigon  | `docker-compose -f erigon.yml up -d`    |
| Geth (suitable for Reth as well) | `docker-compose -f geth.yml up -d`     |
| Geth Clique    | `docker-compose -f geth-clique-consensus.yml up -d`    |
| Nethermind, OpenEthereum    | `docker-compose -f nethermind.yml up -d`    |
| Anvil    | `docker-compose -f anvil.yml up -d`    |
| HardHat network    | `docker-compose -f hardhat-network.yml up -d`    |
| Reef backend only (API)    | `docker-compose -f backend-only.yml up -d`    |

- Running only explorer without DB: `docker-compose -f external-db.yml up -d`. In this case, no db container is created. And it assumes that the DB credentials are provided through `DATABASE_URL` environment variable on the backend container.
- Running explorer with external backend: `docker-compose -f external-backend.yml up -d`
- Running explorer with external frontend: `FRONT_PROXY_PASS=http://host.docker.internal:3000/ docker-compose -f external-frontend.yml up -d`
- Running only the hosted Reef frontend locally: `docker compose -p reef-hosted-frontend -f reef-hosted-frontend.yml up -d`
- Running all microservices: `docker-compose -f microservices.yml up -d`
- Running only explorer without microservices: `docker-compose -f no-services.yml up -d`

All of the built-in local client configs assume the Ethereum JSON RPC is running at http://localhost:8545. The dedicated `backend-only.yml` file is preconfigured for the Reef RPC listed below.

## Backend-only API for Reef

If you only need the Blockscout backend API for Reef, use the dedicated backend-only compose file:

```bash
cd ./docker-compose
docker compose -f backend-only.yml up -d
```

This starts only:

- Postgres
- Redis
- Blockscout backend

The backend API will be available directly at:

- `http://localhost:4000/api/v2/...`

Examples:

- `http://localhost:4000/api/v2/addresses/<address>/transactions`
- `http://localhost:4000/api/v2/addresses/<address>`

Notes:

- The default Reef RPC is `http://eth.reef-node-reefdevcluster-808c46-72-60-35-83.sslip.io/`.
- `ECTO_USE_SSL=false` is required for the bundled local Postgres container. Managed Postgres services may need `ECTO_USE_SSL=true` instead.
- `DISABLE_REALTIME_INDEXER=true` is the default in `backend-only.yml` because the current Reef RPC URL is HTTP-only. If you later get a working WebSocket RPC URL, set `ETHEREUM_JSONRPC_WS_URL` and `DISABLE_REALTIME_INDEXER=false`.
- `INDEXER_DISABLE_PENDING_TRANSACTIONS_FETCHER=true` is the default in `backend-only.yml` because the current Reef RPC does not expose `txpool_content`, and leaving the fetcher on causes repeated `Method not found` errors plus noisy endpoint health flapping.
- To stop the stack: `docker compose -f backend-only.yml down`

## Dokploy

For Dokploy, prefer `backend-only.yml` because it uses Docker named volumes and `${VAR}` syntax, which works well with Dokploy's Docker Compose deployment flow.

Recommended setup:

1. Create a Docker Compose app in Dokploy.
2. Point it at `docker-compose/backend-only.yml`.
3. Add your environment variables in Dokploy's Environment tab.
4. Add a domain in Dokploy's Domains tab and route it to container port `4000`.

Recommended environment variables for Dokploy:

```env
POSTGRES_DB=blockscout
POSTGRES_USER=blockscout
POSTGRES_PASSWORD=change-me
ECTO_USE_SSL=false
SECRET_KEY_BASE=replace-with-a-long-random-string
ETHEREUM_JSONRPC_HTTP_URL=http://eth.reef-node-reefdevcluster-808c46-72-60-35-83.sslip.io/
ETHEREUM_JSONRPC_TRACE_URL=http://eth.reef-node-reefdevcluster-808c46-72-60-35-83.sslip.io/
CHAIN_ID=13939
COIN=REEF
COIN_NAME=Reef
NETWORK=Reef
SUBNETWORK=Mainnet
DISABLE_REALTIME_INDEXER=true
INDEXER_DISABLE_PENDING_TRANSACTIONS_FETCHER=true
```

Once deployed, your API base URL will be:

- `https://your-blockscout-domain/api/v2/...`

In order to stop launched containers, run `docker-compose -f config_file.yml down`, replacing `config_file.yml` with the file name of the config which was previously launched.

You can adjust BlockScout environment variables:

- for backend in `./envs/common-blockscout.env`
- for frontend in `./envs/common-frontend.env`
- for stats service in `./envs/common-stats.env`
- for visualizer in `./envs/common-visualizer.env`
- for user-ops-indexer in `./envs/common-user-ops-indexer.env`

Descriptions of the ENVs are available

- for [backend](https://docs.blockscout.com/setup/env-variables)
- for [frontend](https://github.com/blockscout/frontend/blob/main/docs/ENVS.md).

## Hosted Reef frontend only

If your backend and RPC are already hosted elsewhere and you only want to run the frontend container, use `reef-hosted-frontend.yml`.

Default values in this file are preconfigured for:

- Backend API host: `reef-explorer-ipwkoo-6aeed5-72-60-35-83.sslip.io`
- RPC URL: `http://eth.reef-node-reefdevcluster-808c46-72-60-35-83.sslip.io`
- Chain ID: `13939`

Run it locally:

```bash
cd ./docker-compose
docker compose -p reef-hosted-frontend -f reef-hosted-frontend.yml up -d
```

The frontend will be available on `http://localhost:3000`.

Important variables you may still want to override:

- `NEXT_PUBLIC_APP_HOST` for the frontend's own public hostname
- `NEXT_PUBLIC_APP_PROTOCOL` for the frontend's own public protocol
- `NEXT_PUBLIC_STATS_API_HOST` if your stats service is exposed on a different URL
- `NEXT_PUBLIC_VISUALIZE_API_HOST` if your visualizer service is exposed on a different URL

Example:

```bash
cd ./docker-compose
NEXT_PUBLIC_APP_HOST=reef-frontend.example.com \
NEXT_PUBLIC_APP_PROTOCOL=https \
docker compose -p reef-hosted-frontend -f reef-hosted-frontend.yml up -d
```

### Dokploy

For Dokploy, use `reef-hosted-frontend.dokploy.yml`. It uses `expose: 3000` instead of publishing a host port, which fits Dokploy's routing model better.

Recommended flow:

1. Create a new Docker Compose app in Dokploy using your Git repository, or use the Raw provider and paste the file contents.
2. Point Dokploy at `docker-compose/reef-hosted-frontend.dokploy.yml`.
3. In Dokploy `Environment`, set at least:
   - `NEXT_PUBLIC_APP_HOST=<your frontend domain>`
   - `NEXT_PUBLIC_APP_PROTOCOL=https` if Dokploy serves the site over TLS
4. In Dokploy `Domains`, add your frontend domain and route it to container port `3000`.
5. Deploy.

Notes:

- Dokploy writes UI environment variables to a `.env` file. This compose file references `${...}` variables directly, so Dokploy will substitute them during deployment.
- If the frontend is served over `https`, your browser will block calls to `http` backend or RPC endpoints as mixed content. In that case, switch the backend and RPC URLs to `https` as well, or serve the frontend over plain `http`.

## Running Docker containers via Makefile

Prerequisites are the same, as for docker-compose setup.

Start all containers:

```bash
cd ./docker
make start
```

Stop all containers:

```bash
cd ./docker
make stop
```

***Note***: Makefile uses the same .env files since it is running docker-compose services inside.
