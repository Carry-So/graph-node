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

Subgraph example: https://github.com/gzliudan/bad-token-subgraph  
Query URL: http://<IP>:8000/subgraphs/name/gzliudan/bad-token-subgraph-apothem  
Query command:

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

This subgraph example is deployed on subgraph hosted service: https://thegraph.com/hosted-service/subgraph/gzliudan/bad-token-mumbai also.
