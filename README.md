# EOSIO Contract API
The aim of this project is to provide a wrapper for an API and filler of specific
contracts on eosio based blockchains. It communicates with the eosio State History
Plugin and uses PostgreSQL to store the data. This combination makes it possible to
guarantee that the state of the database matches the onchain state by using table deltas
and action traces. This consistency is achieved with per block Postgres transactions and
internal fork handling which still will be consistent even if you kill the process at 
any time.

## Requirements

* NodeJS >= 14.0
* PostgreSQL >= 12.2
* Redis >= 5.0
* Nodeos >= 1.8.0 (only tested with 2.0) The state history plugin needs to be enabled and the options: 
`trace-history = true`, `chain-state-history = true`

Additional Tools
* Hasura GraphQL Engine >= 1.2 (if you want to allow GraphQL queries)
* PGAdmin 4 (Interface to manage the postgres database)

## Configuration
The config folder contains 3 different configuration files

#### connections.config.json
This file contains Postgres / Redis / Nodeos configuration for the used chain.

Notes
* Redis: Can be used for multiple chains without further action
* PostgreSQL: Each chain needs it own postgres database, but multiple readers of the same
chain can use the same database
* Nodeos: It can be initialized with a snapshot

```javascript
{
  "postgres": {
    "host": "eosio-contract-api-postgres",
    "port": 5432,
    "user": "username",
    "password": "changeme",
    "database": "eosio-contract-api"
  },
  "redis": {
    "host": "eosio-contract-api-redis",
    "port": 6379
  },
  "chain": {
    "name": "wax-testnet",
    "chain_id": "f16b1833c747c43682f4386fca9cbb327929334a762755ebec17f6f23c9b8a12",
    "http": "http://nodeos:8888",
    "ship": "ws://nodeos:8080"
  }
}
```

#### readers.config.json
This file is used to configure the filler

```javascript
[
  // Multiple Readers can be defined and each one will run in a seperated thread
  {
    "name": "atomic", // Name of the reader. Should be unique per chain and should not change after it was started

    "start_block": 0, // start at a specific block
    "stop_block": 0, // stop at a specific block
    "irreversible_only": false, // If you need data for a lot of contracts and do not need live data, this option is faster

    "ship_prefetch_blocks": 50, // How much unconfirmed blocks ship will send
    "ship_min_block_confirmation": 30, // After how much blocks the reader will confirm the blocks

    "delete_data": false, // Truncate all rows which were created by these readers

    "contracts": [
      // AtomicAssets handler which provides data for the AtomicAssets NFT standard
      {
        "handler": "atomicassets",
        "start_on": 100, // Define the block after which actions and deltas are important
        "args": {
          "atomicassets_account": "assetstest55" // Account where the contract is deployed
        }
      }
    ]
  }
]
```

#### server.config.json

```javascript
{
  "provider_name": "pink.network", // Provider which is show in the endpoint documentation
  "provider_url": "https://pink.network",

  "server_addr": "0.0.0.0", // Server address to bind to
  "server_name": "wax.api.atomicassets.io", // Server name which is shown in the documentation
  "server_port": 9000, // Server Port

  "cache_life": 1, // GET endpoints are cached for this amount of time (in seconds)
  "trust_proxy": true, // Enable if you use a reverse proxy to have correct rate limiting by ip

  "rate_limit": {
    "interval": 60, // Interval to reset the counter (in seconds)
    "requests": 240 // How much requests can be made in the defined interval
  },

  "socket_limit": {
    "connections_per_ip": 25, // How much socket connections each IP can have at the same time
    "subscriptions_per_connection": 200 // Subscription limit for each IP
  },

  "ip_whitelist": [], // These IPs are not rate limited or receive cached requests

  "namespaces": [
    // atomicassets namespace which provides an API for basic functionalities
    {
      "name": "atomicassets", 
      "path": "/atomicassets", // Each API endpoint will start with this path
      "args": {
        "atomicassets_account": "atomicassets" // Account where the contract is deployed
      }
    }
  ]
}

```

## Installation

This API consists of two separated processes which need to be started and stopped independently:
* The API which will provide the socket and REST endpoints (or whatever the namespace uses)
* The Filler which will read the data from the blockchain and fill the database

There are two suggested ways to run the project: Docker if you want to containerize the application or PM2 if you want to run it on system level

### Docker

1. `git clone && cd eosio-contract-api`
2. Create and modify configs
3. There is an example docker compose file provided
4. `docker-compose up -d`

#### Start
* `docker-compose start eosio-contract-api-filler`
* `docker-compose start eosio-contract-api-server`

#### Stop
* `docker-compose stop eosio-contract-api-filler`
* `docker-compose stop eosio-contract-api-server`

### PM2

1. `git clone && cd eosio-contract-api`
2. Create and modify configs
3. `yarn install`
4. `yarn global add pm2`

#### Start
* `pm2 start filler`
* `pm2 start api`

#### Stop
* `pm2 stop filler`
* `pm2 stop api`
