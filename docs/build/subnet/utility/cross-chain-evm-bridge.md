---
tags: [Build, Subnets]
description: This tutorial demonstrates the process of deploying a cross-chain bridge between two EVM chains. Build at your own risk.
sidebar_label: Add a Cross-Chain Bridge
pagination_label: Deploy a Cross-Chain EVM Bridge
sidebar_position: 2
---

# Deploy a Cross-Chain EVM Bridge

:::warning

This tutorial is for demo purpose on how to build a cross-chain bridge. It is not for production use.
You must take the full responsibility to ensure your bridge's security.

:::

## Introduction

In this tutorial, we will be building a bridge between **[WAGMI](/build/subnet/info/wagmi.md)** and
**[Fuji](/learn/avalanche/fuji.md)**. This bridge will help us to transfer native **WGM** coin
wrapped into **wWGM** back and forth from the WAGMI chain to the Fuji chain. Using this guide, you
can deploy a bridge between any EVM-based chains for any ERC20 tokens.

The wrapped version of a native coin is its pegged ERC20 representation. Wrapping it with the ERC20
standard makes certain processes like delegated transactions much easier. You can easily get wrapped
tokens by sending the native coin to the wrapped token contract address.

> WAGMI is an independent EVM-based test chain deployed on a custom Subnet on the Avalanche network.

We will be using **ChainSafe**'s bridge repository, to easily set up a robust and secure bridge.

## Workflow of the Bridge

WAGMI and Fuji chains are not interconnected by default, however, we could make them communicate.
Relayers watch for events (by polling blocks) on one chain and perform necessary action using those
events on the other chain. This way we can also perform bridging of tokens from one chain to the
other chain through the use of smart contracts.

Here is the basic high-level workflow of the bridge -

- Users deposit token on the Bridge contract
- Bridge contract asks Handler contract to perform deposit action
- Handler contract **locks** the deposited token in the token safe
- Bridge contract emits `Deposit` event
- Relayer receives the `Deposit` event from the source chain
- Relayer creates a voting proposal on the destination chain to mint a new token
- After threshold relayer votes, the proposal is executed
- Tokens are **minted** to the recipient's address

Bridging tokens from source to destination chain involves the **lock and mint** approach. Whereas
bridging tokens from destination to source chain involves **burn and release** approach. We cannot
mint and burn tokens that we do not control. Therefore we lock them in the token safe on the source
chain. And mint the corresponding token (which we will deploy and hence control) on the destination
chain.

![architecture](/img/chainsafe-bridge-1-workflow.png)

## Requirements

These are the requirement to follow this tutorial -

- Set up [WAGMI](/build/subnet/info/wagmi.md#adding-wagmi-to-core) and
[Fuji](/build/dapp/fuji-workflow.md#set-up-fuji-network-on-core-optional) on Core
- Import `wWGM` token (asset) on the WAGMI network (Core). Here is the address - `0x3Ee7094DADda15810F191DD6AcF7E4FFa37571e4`
- `WGM` coins on the WAGMI chain. Drip `1 WGM` from the [WAGMI Faucet](https://faucet.trywagmi.xyz/).
- `AVAX` coins on the Fuji chain. Drip `10 AVAX` from the [Fuji Faucet](https://faucet.avax.network/).
If you already have an AVAX balance greater than zero on Mainnet, 
paste your C-Chain address there, and request test tokens. Otherwise, 
please request a faucet coupon on 
[Discord](https://discord.com/channels/578992315641626624/1193594716835545170).
- Wrapped `WGM` tokens on the WAGMI chain. Send a few `WGM` coins to the `wWGM` token address (see
second point), to receive the same amount of `wWGM`. Always keep some `WGM` coins, to cover transaction
fees.

## Setting Up Environment

Let's make a new directory `deploy-bridge`, where we will be keeping our bridge codes. We will be
using the following repositories -

- [`ChainSafe/chainbridge-deploy`](https://github.com/ChainSafe/chainbridge-deploy) - This will help
us in setting up of our bridge contracts
- [`ChainSafe/ChainBridge`](https://github.com/ChainSafe/ChainBridge) - This will help us in setting
up of our off-chain relayer.

### Installing ChainBridge Command-Line Tool

Using the following command, we can clone and install ChainBridge's command-line tool. This will
help us in setting up bridge contracts and demonstrating bridge transfers. Once the bridge contracts
are deployed, you can use its ABI and contract address to set up your UI.

```bash
git clone -b v1.0.0 --depth 1 https://github.com/ChainSafe/chainbridge-deploy \
&& cd chainbridge-deploy/cb-sol-cli \
&& npm install \
&& make install
```

This will build the contracts and installs the `cb-sol-cli` command.

### Setting Up Environment Variables

Let's set up environment variables, so that, we do not need to write their values every time we
issue a command. Move back to the `deploy-bridge` directory (main project directory) and make a
new file `configVars`. Put the following contents inside it -

```env
SRC_GATEWAY=https://subnets.avax.network/wagmi/wagmi-chain-testnet/rpc
DST_GATEWAY=https://api.avax-test.network/ext/bc/C/rpc

SRC_ADDR="<Your address on WAGMI>"
SRC_PK="<your private key on WAGMI>"
DST_ADDR="<Your address on Fuji>"
DST_PK="<your private key on Fuji>"

SRC_TOKEN="0x3Ee7094DADda15810F191DD6AcF7E4FFa37571e4"
RESOURCE_ID="0x00"
```

- `SRC_ADDR` and `DST_ADDR` are the addresses that will deploy bridge contracts and will act as a relayer.
- `SRC_TOKEN` is the token that we want to bridge. Here is the address of the wrapped ERC20 version
of the WGM coin aka wWGM.
- `RESOURCE_ID` could be anything. It identifies our bridged ERC20 tokens on both sides (WAGMI and Fuji).

Every time we make changes to these config variables, we have to update our bash environment. Run
the following command according to the relative location of the file. These variables are temporary
and are only there in the current terminal session, and will be flushed, once the session is over.
Make sure to load these environment variables anywhere you will using them in the bash commands
(like `$SRC_GATEWAY` or `$SRC_ADDR`)

```bash
source ./configVars
```

## Setting Up Source Chain

We need to set up our source chain as follows -

- Deploy Bridge and Handler contract with `$SRC_ADDR` as default and only relayer
- Register the `wWGM` token as a resource on the bridge

### Deploy Source Contracts

The command-line tool `cb-sol-cli` will help us to deploy the contracts. Run the following command
in the terminal session where the config vars are loaded. It will add `SRC_ADDR` as the default
relayer for relaying events from the WAGMI chain (source) to the Fuji chain (destination).

**One of the most important parameter to take care of while deploying bridge contract is the `expiry`**
**value. It is the number of blocks after which a proposal is considered cancelled. By default it is**
**set to `100`. On Avalanche Mainnet, with this value, the proposals could be expired within 3-4 minutes.**
**You should choose a very large expiry value, according to the chain you are deploying bridge to.**
**Otherwise your proposal will be cancelled if the threshold number of vote proposals are not received**
**on time.**

You should also keep this in mind that sometimes during high network activity, a transaction could
stuck for a long time. Proposal transactions stuck in this scenario, could result in the cancellation
of previous proposals. Therefore, expiry values should be large enough, and relayers should issue
transactions with a competitive max gas price.

```bash
cb-sol-cli --url $SRC_GATEWAY --privateKey $SRC_PK --gasPrice 25000000000 deploy \
    --bridge --erc20Handler \
    --relayers $SRC_ADDR \
    --relayerThreshold 1 \
    --expiry 500 \
    --chainId 0
```

The output will return deployed contracts' (Bridge and Handler) address. Update the `configVars`
file with these addresses by adding the following 2 variables and loading them to the environment.

```env
SRC_BRIDGE="<resulting bridge contract address>"
SRC_HANDLER="<resulting erc20 handler contract address>"
```

Make sure to load these using the `source` command.

### Configure Resource on Bridge

Run the following command to register the `wWGM` token as a resource on the source bridge.

```bash
cb-sol-cli --url $SRC_GATEWAY --privateKey $SRC_PK --gasPrice 25000000000 bridge register-resource \
    --bridge $SRC_BRIDGE \
    --handler $SRC_HANDLER \
    --resourceId $RESOURCE_ID \
    --targetContract $SRC_TOKEN
```

## Setting Up Destination Chain

We need to set up our destination chain as follows -

- Deploy Bridge and Handler contract with `$DST_ADDR` as default and only relayer
- Deploy mintable and burnable ERC20 contract representing bridged `wWGM` token
- Register the `wWGM` token as a resource on the bridge
- Register the` wWGM` token as mintable/burnable on the bridge
- Giving permissions to Handler contract to mint new `wWGM` tokens

### Deploy Destination Contracts

Run the following command to deploy Bridge, ERC20 Handler, and `wWGM` token contracts on the Fuji
chain. Again it will set `DST_ADDR` as the default relayer for relaying events from Fuji chain
(destination) to WAGMI chain (source). For this example, both `SRC_ADDR` and `DST_ADDR` represent
the same thing.

```bash
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 25000000000 deploy\
    --bridge --erc20 --erc20Handler \
    --relayers $DST_ADDR \
    --relayerThreshold 1 \
    --chainId 1
```

Update the environment variables with the details which you will get by running the above command.
Don't forget to load these variables.

```env
DST_BRIDGE="<resulting bridge contract address>"
DST_HANDLER="<resulting erc20 handler contract address>"
DST_TOKEN="<resulting erc20 token address>"
```

### Configuring Resource on Bridge

Run the following command to register deployed `wWGM` token as a resource on the bridge.

```bash
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 25000000000 bridge register-resource \
    --bridge $DST_BRIDGE \
    --handler $DST_HANDLER \
    --resourceId $RESOURCE_ID \
    --targetContract $DST_TOKEN
```

### Setting Token as Mintable and Burnable on Bridge

The bridge has two options when it receives a deposit of a token -

- Lock the received token on one chain and mint the corresponding token on the other chain
- Burn the received token on one chain and release the corresponding token on the other chain

We cannot mint or burn any token which we do not control. Though we can lock and release such tokens
by putting them in a token safe. The bridge has to know which token it can burn. With the following
command, we can set the resource as burnable. The bridge will choose the action accordingly, by
seeing the token as burnable or not.

```bash
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 25000000000 bridge set-burn \
    --bridge $DST_BRIDGE \
    --handler $DST_HANDLER \
    --tokenContract $DST_TOKEN
```

### Authorizing Handler to Mint New Tokens

Now let's permit the handler to mint the deployed ERC20 (wWGM) token on the destination chain. Run
the following command.

```bash
cb-sol-cli --url $DST_GATEWAY --privateKey $DST_PK --gasPrice 25000000000 erc20 add-minter \
    --minter $DST_HANDLER \
    --erc20Address $DST_TOKEN
```

> **The deployer of the contracts (here `SRC_ADDR` or `DST_ADDR`) holds the admin rights. An admin**
**can add or remove a new relayer, minter, admin etc. It can also mint new ERC20 tokens on the**
**destination chain. You can issue these commands using `cb-sol-cli` with the options mentioned in**
**these [files](https://github.com/ChainSafe/chainbridge-deploy/tree/main/cb-sol-cli/docs). The mint**
**command should not be used manually, unless some intervention is required, when the relayers failed**
**to mint the tokens on the destination chain on time.**

## Deploy Relayer

All the on-chain setups like deploying bridges, handlers, tokens, etc. are complete. But the two
chains are not interconnected. We need some off-chain relayer to communicate messages between the
chains. The relayer will poll for deposit events on one chain, and submit vote proposals to mint or
release the corresponding token on another chain.

Since we set the relayer threshold to 1, while deploying the bridge and handler, we require a voting
proposal from only 1 relayer. But in production, we should use a large set of relayers with a high
threshold to avoid power concentration.

For this purpose, we will be using ChainSafe's relayer. Follow the steps described below to deploy
the relayer.

### Cloning and Building Relayer

Open a new terminal session, while keeping the previous session loaded with environment variables.
We have to load the environment variables in this session too. Load these variables in this session
too using the `source` command.

Now, move to the `deploy-bridge` directory and run the following command to clone the relayer repository
(implemented in Go), and build its binary.

```bash
git clone -b v1.1.1 --depth 1 https://github.com/ChainSafe/chainbridge \
&& cd chainbridge \
&& make build
```

This will create a binary inside the `chainbridge/build` directory as `chainbridge`.

### Configuring Relayer

The relayer requires some configurations like source chain, destination chain, bridge, handler
address, etc. Run the following command. It will make a `config.json` file with the required
details in it. You can update these details, as per your need.

```bash
echo "{
  \"chains\": [
    {
      \"name\": \"WAGMI\",
      \"type\": \"ethereum\",
      \"id\": \"0\",
      \"endpoint\": \"$SRC_GATEWAY\",
      \"from\": \"$SRC_ADDR\",
      \"opts\": {
        \"bridge\": \"$SRC_BRIDGE\",
        \"erc20Handler\": \"$SRC_HANDLER\",
        \"genericHandler\": \"$SRC_HANDLER\",
        \"gasLimit\": \"1000000\",
        \"maxGasPrice\": \"50000000000\",
        \"http\": \"true\",
        \"blockConfirmations\":\"0\"
      }
    },
    {
      \"name\": \"Fuji\",
      \"type\": \"ethereum\",
      \"id\": \"1\",
      \"endpoint\": \"$DST_GATEWAY\",
      \"from\": \"$DST_ADDR\",
      \"opts\": {
        \"bridge\": \"$DST_BRIDGE\",
        \"erc20Handler\": \"$DST_HANDLER\",
        \"genericHandler\": \"$DST_HANDLER\",
        \"gasLimit\": \"1000000\",
        \"maxGasPrice\": \"50000000000\",
        \"http\": \"true\",
        \"blockConfirmations\":\"0\"
      }
    }
  ]
}" >> config.json
```

Check and confirm the details in the `config.json` file.

> In the above command, you can see that `blockConfirmations` is set to `0`. This will work well for
networks like Avalanche because the block is confirmed once it's committed. Unlike other chains such
 as Ethereum, which requires 20-30 block confirmations. Therefore, use this configuration with
 caution, depending on the type of chain you are using.
>
> It can cause serious problems if a corresponding token is minted or released based on an
unconfirmed block.

### Set Up Keys

Give relayer access to your keys. Using these keys, the relayer will propose deposit events and
execute proposals. It will ask to set a password for encrypting these keys. Every time you start
the relayer, it will ask for this password.

```bash
./build/chainbridge accounts import --privateKey $SRC_PK
```

```bash
./build/chainbridge accounts import --privateKey $DST_PK
```

## Let's Test the Bridge

The setup is now complete - both on-chain and off-chain. Now we just have to start the relayer and
test the bridge. For testing purposes, we will be using `cb-sol-cli` to make deposit transactions on
the bridge. But you can make your frontend and integrate it with the bridge using the ABIs.

### Start Relayer

Run the following command to start the relayer. It will print logs of all the events associated with
our bridge, happening on both the chains. So keep the relayer running and follow the next commands
in the other terminal session.

```bash
./build/chainbridge --config config.json --verbosity trace --latest
```

### Approve Handler to Spend my Tokens

Now, let's deposit tokens on the WAGMI bridge. But before that, we need to approve the handler to
spend (lock or burn) tokens on our (here `SRC_PK`) behalf. The amount here is in Wei (1 ether (WGM)
= 10^18 Wei). We will be approving 0.1 wWGM.

```bash
cb-sol-cli --url $SRC_GATEWAY --privateKey $SRC_PK --gasPrice 25000000000 erc20 approve \
    --amount 100000000000000000 \
    --erc20Address $SRC_TOKEN \
    --recipient $SRC_HANDLER
```

### Deposit Tokens to the Bridge

Once approved, we can send a deposit transaction. Now let's deposit 0.1 wWGM on the bridge. The
handler will lock (transfer to token safe) 0.1 wWGM from our address (here `SRC_PK`) and mint the
new tokens on the destination chain to the recipient (here `DST_ADDR`).

```bash
cb-sol-cli --url $SRC_GATEWAY --privateKey $SRC_PK --gasPrice 25000000000 erc20 deposit \
    --amount 100000000000000000 \
    --dest 1 \
    --bridge $SRC_BRIDGE \
    --recipient $DST_ADDR \
    --resourceId $RESOURCE_ID
```

This transaction will transfer 0.1 wWGM to token safe and emit a `Deposit` event, which will be
captured by the relayer. Following this event, it will send a voting proposal to the destination
chain. Since the threshold is 1, the bridge will execute the proposal, and new wWGM minted to the
recipient's address. Here is the screenshot of the output from the relayer.

![output](/img/chainsafe-bridge-2-relayer-output.png)

Similarly, we can transfer the tokens back to the WAGMI chain.

## Conclusion

Similar to the above process, you can deploy a bridge between any 2 EVM-based chains. We have used
the command-line tool to make approvals and deposits. This can be further extended to build a
frontend integrated with the bridge. Currently, it depends on a single relayer, which is not secure.
We need a large set of relayers and a high threshold to avoid any kind of centralization.

You can learn more about these contracts and implementations by reading ChainSafe's
[ChainBridge](https://chainbridge.chainsafe.io/) documentation.
