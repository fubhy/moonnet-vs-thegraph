# MoonNet & TheGraph Experimentation

## Problem #1: Genesis block fetching [FIXED]

### Description

TheGraph is querying for the genesis block using `earliest` as formally specified in the RPC spec: https://github.com/ethereum/wiki/wiki/JSON-RPC#parameters-27

This is not supported by ganache (and others).

### Fix

As the spec is not followed strictly across the board (web3, ganache, ethers, etc. all don't follow it fully and usually only support `latest`), it's probably best to simply use `0x0` in graph-node.

Pull request: https://github.com/graphprotocol/graph-node/pull/1658

## Problem #2: Indexing exceeding archive node rate limit

### Description

When indexing smart contracts deployed into the (local) fork and configuring the `startBlock` in the `subgraph.yaml` to the correct block
height (within the boundaries of the fork) at which the contract was deployed, the `graph-node` indexing still reaches outside of the configured block height, causing the underlying archive node to eventually lock us out due to us exceeding the rate limit.

### Steps to reproduce

1. Clone this repository
2. Copy the `.env.example` file to `.env` and fill in the `MOONET_UUID` parameter
3. Run `docker-compose pull` to make sure you are on the correct docker image versions used in `docker-compose.yaml`
4. Run `docker-compose up -d` to boot the docker services.

> **NOTE**: Make sure that the used ports in the `docker-compose.yaml` port mappings are not occupied by e.g. a locally running Ganache instance.

5. Run `(cd subgraph && yarn)`
6. Run `(cd subgraph && yarn truffle migrate)`

> **NOTE**: This already fails frequently for me. Already here, I would expect the hosted Ganache fork to not even be involved in the deployment because I am deploying into my local fork (fork of fork). This assumption doesn't seem to be correct because I am seeing lots of calls to `eth_getBlockByNumber` and `eth_getTransactionByHash` in my MoonNet logs.

6. Copy & paste the `GravatarRegistry` contract address and block number at which it was deployed to `subgraph/subgraph.yaml`

> **NOTE**: You can find both in the log output of the `truffle migrate` command. The `startBlock` configuration option should result in `graph-node` only querying and indexing block heights above this threshold. I am therefore not expecting any calls to fall through to the underlying archive node. In fact, it _shouldn't_ even hit the hosted MoonNet Ganache fork (but it does).

7. Run `(cd subgraph && yarn codegen && yarn create-local || true && yarn deploy-local)`

> **NOTE**: For subsequent runs, you actually only need to run `yarn deploy-local` but the above command runs them all in sequence for convenience.

#### Testing

You should now be able to navigate to http://127.0.0.1:8000/subgraphs/name/fubhy/moonnet to see the `graphiql` interface. If all worked, you can run the following query:

```gql
{
  gravatars {
    id
    owner
    imageUrl
    displayName
  }
}
```

#### Logs

Run `docker-compose logs -f` to see and follow all logs emitted from the docker containers.

#### Starting over

If, at any point in time you need to start over, simply run `docker-compose down --remove-orphans --volumes` to purge the docker container state and volumes.

#### Working version

If you want to see how this example _should_ work, simply remove the `fork` option from the `ganache` service in `docker-compose.yaml`.
