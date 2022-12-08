# Graph node for xdc blockchain

Original README.md is moved to: [README](./README.original.md).

## 1) deploy from docker

```shell
cd docker
docker compose up
```

## 2) deploy from srouce

[build from srouce step by step](./docs/deploy-from-source.md)

## 3) Query

An example is developed at: http://103.101.129.136:8000/subgraphs/name/gzliudan/bad-token-subgraph-apothem, you can use below query command:

```graphql
{
  erc20Contracts(first: 5) {
    id
    name
    symbol
    decimals
    totalSupply {
      value
      valueExact
    }
  }
  blackLists(first: 5) {
    id
    members {
      account {
        id
      }
    }
  }
  accounts(first: 5) {
    id
    isErc20
    blackLists {
      id
    }
    Erc20balances {
      id
      value
      valueExact
    }
  }
}
```

This example is deployed on subgraph hosted service also: https://thegraph.com/hosted-service/subgraph/gzliudan/bad-token-mumbai.
