<h1 align="center">Trobium</h1>
<h4 align="center">
  Java full node for the Trobium L1
</h4>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-LGPLv3-blue.svg"></a>
</p>

## Table of Contents

- [What’s Trobium?](#whats-trobium)
- [Building the source code](#building-the-source-code)
- [Executables](#executables)
- [Running a node](#running-a-node)
- [Talking to a full node](#talking-to-a-full-node)
- [Contribution](#contribution)
- [Resources](#resources)
- [License](#license)

# What's Trobium?

Trobium is an application-oriented L1. This repository is the Java full node, based on java-tron GreatVoyage **4.8.2**.

| | |
|--|--|
| Coin | **TROB** |
| Atomic unit | **TROBI** (`1 TROB = 1_000_000 TROBI`) |
| Addresses | Prefix `0x41`. Base58Check addresses start with `T` on testnet and mainnet |
| Consensus | DPoS, **21** Super Representatives, **3 second** blocks |
| VM | **TVM** |
| Tokens | **TRB10** (native asset) and **TRB20** (TVM contract tokens) |
| SDK | npm [`trobkit`](https://www.npmjs.com/package/trobkit) |
| Sign prefix | `\x19Trobium Signed Message:\n` |
| HD path | `m/44'/195'/0'/0/0` |

Contract calls use `fee_limit` in TROBI. The network is the node URL. There is no Ethereum `chainId` or `gasPrice`.

Java and protobuf identifiers in this tree still say TRX, sun, and TRC*. Product names above are what wallets, explorers, and `trobkit` use.

# Building the source code

Before building, make sure you have:

- Hardware with at least 4 CPU cores, 16 GB RAM, and 10 GB free disk for a smooth compile.
- Operating system: `Linux` or `macOS` (`Windows` is not supported).
- Git and the JDK that matches your CPU.

| CPU architecture | Required JDK |
| :--------------- | :----------- |
| `x86_64` / `amd64` | JDK 8 |
| `ARM64` / `aarch64` | JDK 17 |

> **Note**: `ARM64` / `aarch64` support starts with GreatVoyage-v4.8.1. The two JDKs are not interchangeable. Mainnet Super Representative hosts in the operator guide run **JDK 8** on Ubuntu 20.04 or 22.04 amd64.

Install dependencies with the bundled script:

```bash
chmod +x install_dependencies.sh
./install_dependencies.sh
```

Then compile from this checkout:

```bash
./gradlew clean build -x test
```

`-x test` skips the test suite. The node jar lands in `build/libs/FullNode.jar`.

# Executables

| Artifact / script | Description |
| :---------------- | :---------- |
| **`FullNode.jar`** | Full node (`build/libs/` after a successful build). `java -jar FullNode.jar --help` lists flags. Add `--witness` to produce blocks. |
| **`Toolkit.jar`** | Node maintenance utility in `build/libs/`: partition, prune, copy, and convert databases. |
| **`start.sh`** | Quick-start helper for x86_64 and JDK 8. See [shell.md](./shell.md). |
| **`start.sh.simple`** | Quick-start template for ARM64 and JDK 17. Usage notes are in the script. |

# Running a node

The `config.conf` shipped in this tree is a stock template. It is **not** the live Trobium genesis or seed list. Join testnet or mainnet with the network config from operations (P2P version `1`, listen port **16665**).

## Hardware

| Role | CPU | Memory | Disk | Network |
| :--- | :-- | :----- | :--- | :------ |
| Full node (sync / RPC) | 8+ cores | 16 GB+ | 3 TB free | — |
| Super Representative | 32+ cores | 64 GB+ | High-performance SSD, 4 TB+ | 100+ MB/s |

The 8-core / 16 GB floor is enough to sync. It is not enough to produce blocks reliably. Testnet can run on less.

## Networks

| | Testnet | Mainnet |
|--|---------|---------|
| Public full node | `https://fullnode-one-testnet.trobchain.com` | Ops `FULLNODE_URL` (HTTP **16667**) |
| Direct HTTP | `http://103.57.64.113:16667` | — |
| Solidity HTTP | `http://103.57.64.113:16668` | **16668** on the node |
| Explorer | [testnet.trobchain.com](https://testnet.trobchain.com) | [trobchain.com](https://trobchain.com) |
| Backend | [testnet-backend.trobchain.com](https://testnet-backend.trobchain.com) (faucet) | [backend.trobchain.com](https://backend.trobchain.com) |

`/wallet/gettotalsupply` exists on the testnet node and is absent on the mainnet tree.

Default ports on a live Trobium node:

| Interface | Port |
| :-------- | :--- |
| P2P (TCP and UDP) | 16665 |
| HTTP full node | 16667 |
| HTTP solidity | 16668 |
| gRPC | 16669 |

### Launch

Point `-c` at the Trobium config for the network you are joining:

```bash
java -jar ./build/libs/FullNode.jar -c /path/to/trobium.conf
```

Follow sync in the node log (the file name in this tree is still `tron.log`):

```bash
tail -f ./logs/tron.log
```

Compare the tip with [testnet.trobchain.com](https://testnet.trobchain.com) or [trobchain.com](https://trobchain.com).

### Super Representative

An SR node is the same jar with `--witness`. It produces a block every 3 seconds when the account is in the top **21**. Put the SR account private key in `localwitness`:

```
localwitness = [
  <your_private_key>
]
```

```bash
java -jar ./build/libs/FullNode.jar -c /path/to/trobium.conf --witness
```

Keep the host on NTP. Slot time is 3000 ms, and a skewed clock misses blocks.

# Talking to a full node

Use **trobkit** against a synced node. Amounts on the wire are integer TROBI.

```ts
import { Trobkit } from 'trobkit';

const trobkit = new Trobkit({
  fullHost: 'https://fullnode-one-testnet.trobchain.com',
  privateKey: process.env.TROB_PRIVATE_KEY,
});
```

Native transfer: `transactionBuilder.sendTrob` → `trob.sign` → `trob.sendRawTransaction` (or `trob.sendTrob`). Contracts: `trobkit.contract(abi, addr)`, then `.call()` / `.send()`. Default `feeLimit` is `150_000_000` TROBI.

The node also exposes HTTP, gRPC, and JSON-RPC. Turn each one on in the config you actually run. Live Trobium remaps the stock `8090` / `8545` / `9090` defaults to the ports in the table above:

```
node {
  http {
    fullNodeEnable = true
    fullNodePort   = 16667
  }

  rpc {
    enable = true
    port   = 16669
  }
}
```

If any of these listen on a public interface, put authentication, rate limits, and network controls in front of them.

# Contribution

See [CONTRIBUTING.md](./CONTRIBUTING.md).

# Resources

- [trobkit](https://www.npmjs.com/package/trobkit) — JavaScript / TypeScript SDK (`sendTrob`, staking, SR votes, TRB20).
- [Testnet explorer](https://testnet.trobchain.com) and [mainnet explorer](https://trobchain.com).
- [Testnet faucet API](https://testnet-backend.trobchain.com).
- [Testnet full node](https://fullnode-one-testnet.trobchain.com).

# License

This node is released under the [LGPLv3 license](./LICENSE). It is a fork of java-tron (GreatVoyage).
