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

- Running only explorer without DB: `docker-compose -f external-db.yml up -d`. In this case, no db container is created. And it assumes that the DB credentials are provided through `DATABASE_URL` environment variable on the backend container.
- Running explorer with external backend: `docker-compose -f external-backend.yml up -d`
- Running explorer with external frontend: `FRONT_PROXY_PASS=http://host.docker.internal:3000/ docker-compose -f external-frontend.yml up -d`
- Running only the hosted Reef frontend locally: `docker compose -p reef-hosted-frontend -f reef-hosted-frontend.yml up -d`
- Running all microservices: `docker-compose -f microservices.yml up -d`
- Running only explorer without microservices: `docker-compose -f no-services.yml up -d`

All of the configs assume the Ethereum JSON RPC is running at http://localhost:8545.

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

- Backend API host: `explorer-backend-sij6uw-1d3882-72-60-35-83.nip.io`
- RPC URL: `https://eth.reef-node-reefdevcluster-808c46-72-60-35-83.nip.io/`
- Chain ID: `13939`

The Dokploy hosted frontend compose file now runs an Nginx edge proxy in front of the frontend container.
The browser talks to the frontend domain for `/api/...` and `/socket/...`, and that proxy forwards those requests to `BACK_PROXY_PASS`.
This avoids mixed-content issues when the frontend is served over `https` and keeps websocket traffic on the frontend origin even when the hosted Reef backend sits behind a separate TLS domain.

Run it locally:

```bash
cd ./docker-compose
docker compose -p reef-hosted-frontend -f reef-hosted-frontend.yml up -d
```

The frontend will be available on `http://localhost:3009`.

Important variables you may still want to override:

- `NEXT_PUBLIC_APP_HOST` for the frontend's own public hostname. Locally it defaults to `localhost:3009`
- `NEXT_PUBLIC_APP_PROTOCOL` for the frontend's own public protocol
- `NEXT_PUBLIC_STATS_API_HOST` only if you actually run a separate Blockscout stats service. Leave it empty to keep `daily_txs` on the built-in `/api/v2/stats/charts/transactions` backend endpoint.
- `NEXT_PUBLIC_VISUALIZE_API_HOST` if your visualizer service is exposed on a different URL
- `NEXT_PUBLIC_USE_NEXT_JS_PROXY=false` to force the frontend to call your hosted backend directly instead of routing through `/node-api/proxy`

Example:

```bash
cd ./docker-compose
NEXT_PUBLIC_APP_HOST=reef-frontend.example.com \
NEXT_PUBLIC_APP_PROTOCOL=https \
FRONTEND_PORT=3010 docker compose -p reef-hosted-frontend -f reef-hosted-frontend.yml up -d
```

### Dokploy

For Dokploy, use `reef-hosted-frontend.dokploy.yml`. It exposes the `proxy` service on container port `80`, which fits Dokploy's routing model better.

Recommended flow:

1. Create a new Docker Compose app in Dokploy using your Git repository, or use the Raw provider and paste the file contents.
2. Point Dokploy at `docker-compose/reef-hosted-frontend.dokploy.yml`.
3. In Dokploy `Environment`, set at least:
   - `FRONTEND_PUBLIC_HOST=<your frontend domain>`
   - `NEXT_PUBLIC_APP_PROTOCOL=https` if Dokploy serves the site over TLS
   - `BACK_PROXY_PASS=https://explorer-backend-sij6uw-1d3882-72-60-35-83.nip.io`
4. In Dokploy `Domains`, add your frontend domain and route it to container port `80`.
5. Deploy.

Notes:

- Dokploy writes UI environment variables to a `.env` file. This compose file references `${...}` variables directly, so Dokploy will substitute them during deployment.
- `FRONTEND_PUBLIC_HOST` should match the exact public hostname users open in the browser. The Dokploy file uses it for the frontend's own API and websocket origin so requests stay on the same TLS domain.
- The local compose file keeps the simpler direct-browser-to-backend setup and is intended for `http://localhost:3009`.
- The Dokploy compose file keeps `NEXT_PUBLIC_USE_NEXT_JS_PROXY=false`, but it places an Nginx proxy in front of the frontend container. The browser uses the frontend domain for `/api` and `/socket`, and Nginx forwards those requests to `BACK_PROXY_PASS`.
- Both hosted frontend compose files now leave `NEXT_PUBLIC_STATS_API_HOST` empty by default. That is intentional for this Reef setup: it prevents the frontend from switching into separate stats-service mode and keeps the `Daily transactions` chart on the working backend route `/api/v2/stats/charts/transactions`.
- The local compose file still defaults `FRONTEND_PORT` to `3009`, and the frontend container itself also listens on that same port.
- The Dokploy compose file keeps the frontend container on internal port `3000` and exposes the Nginx proxy on port `80`.
- This proxy setup fixes the frontend websocket mixed-content problem by terminating browser traffic on the frontend domain and forwarding `/socket/...` upstream to the hosted backend from inside Docker.

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
