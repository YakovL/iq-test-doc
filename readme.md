IQ SDK makes it simple to stake, rent or swap tokens using services of various enterprises. <!-- closer to business? more features?

To use a service, you'll need:

* the network's RPC endpoint and the client's address and private key
* the enterprise address <!-- where to get from? (no listing available)
* the service address <!-- where to get from?

Installing IQ SDK
-----------------
The SDK consists of [several packages](https://iqlabsorg.github.io/iq-sdk-js/) with [sources in a monorepo](https://github.com/iqlabsorg/iq-sdk-js/).

To get SDK features for a specific blockchain, one needs a provider for that blockchain.
For now, a single generic `EIP155BlockchainProvider` that supports all EVM compatible blockchains
is available in the `@iqprotocol/eip155` package. The SDK is extendable and other providers can be created in the future.

Let's install dependencies for fetching some info:

```
npm i @iqprotocol/eip155  ethers typescript ts-node @types/node
```

and import the provider:

```typescript
import { EIP155BlockchainProvider } from '@iqprotocol/eip155';
```

Preparing a signer <!-- Wallet – correct signer? -->
------------------

To instantiate a blockchain provider, we need a signer that'll be passed like this:

```typescript
const blockchain = new EIP155BlockchainProvider({
    signer,
});
```

so let's create one. The signer contains info about the chain that we're going to use
and about the client. For instance, if we'd like to get info about an enterprise
in BSC Testnet (a "read-only" method), we have to [find](https://docs.ricefarm.fi/guides/metamask-add-bsc)
that its RPC endpoint is `https://data-seed-prebsc-1-s1.binance.org:8545/`, take some valid network address
(one of ours or just from an explorer) and it's enough to create a `VoidSigner`:

```typescript
import { VoidSigner, providers } from 'ethers';

const provider = new providers.JsonRpcProvider('https://data-seed-prebsc-1-s1.binance.org:8545/');
const clientAddress = '0x049153b8DAe0a232Ac90D20C78f1a5D1dE7B7dc5';
const signer = new VoidSigner(clientAddress, provider);
```

To check the connection we can try to get the chain id:

```typescript
blockchain.getChainId().then(({ reference }) => console.log(`chain id is ${reference}`));
```

For a "write" method (like swap, stake or rent), another type of signer is required, for instance a Wallet:

```typescript
import { Wallet, providers } from 'ethers';

const provider = new providers.JsonRpcProvider('https://data-seed-prebsc-1-s1.binance.org:8545/');
const clientPrivateKey = '0xf3921efdfe42ea58356da0ab453ac073b6ab7a4a58f20aebfa408bbd57a91ee8';
const signer = new Wallet(clientPrivateKey, provider);
```

Learning about enterprises and services <!-- where can we find that address? (Parsiq enterprise in BSC Testnet) *serviceFeePercent = ? -->
---------------------------------------
Now that we have a signer and can connect to the network, we can interact with an enterprise.

First, let's get some info about it. For example, for Parsiq enterprise in BSC Testnet we can call

```typescript
(async() => {
    const enterpriseInfo = await blockchain.enterprise(enterpriseAddress).getInfo();
    console.log(enterpriseInfo);

    const servicesAddresses = await blockchain.enterprise(enterpriseAddress).getServiceAddresses();
    console.log(servicesAddresses);

    for(const address of servicesAddresses) {
        console.log((await blockchain.service(address).getInfo()));
    }
})();
```

to get its metadata, services' addresses and metadata. This also gives some idea on how to use the "write" methods.

Stake, rend and swap
--------------------

With a wallet signer, we can do various things with client's funds.

### Staking

Staking means providing client's funds to an enterprise's pool to get a reward later.
<!-- how reward is calculated? is this like investing? can an enterprise go bankrupt? -->

There's a number of `Enterprise` methods to control staking. Let's stake something first:

<!-- _amount: what units? (what token?) -->
```typescript
const _amount = 100;
const tx = await blockchain.enterprise(enterpriseAddress).stake(_amount);
```

As a result, we get a stake token (an NFT) which holds the staked amount and other info. Once we get it, we can call
`Enterprise.getStakeTokenIds` to get the id of the stake token(s). To find out the stake amount, we can use
`Enterprise.getStake(tokenId)`. There's also `increaseStake`, `decreaseStake` and `unstake` to manipulate the amount,
several metadata getters. Finally, there's `getStakingReward`, `claimStakingReward` and `setEnterpriseTokenAllowance`.
<!-- what does setEnterpriseTokenAllowance do? -->

### Renting

___

### Swapping

___

Statuses and fees
-----------------

___fee in ContractTransaction?
___estimating fees?
___tx status in ContractTransaction?

Wrapping up
-----------

___