IQ SDK makes it simple to build applications powered by IQ Protocol.
Interactions with enterprises can be quite complex; this article explains the basics of three interactions:
staking into an enterprise pool and getting enterprise services' tokens via swapping or renting.
Such tokens give various possibilities, like opening access to certain APIs etc.

To use a service, you'll need:

* the network's RPC endpoint and the client's address and private key
* the enterprise address
* the service address
* enterprise tokens (for staking and usually for renting and swapping)

Installing IQ SDK
-----------------
The SDK consists of [several packages](https://iqlabsorg.github.io/iq-sdk-js/) with [sources in a monorepo](https://github.com/iqlabsorg/iq-sdk-js/).

To get SDK features for a specific blockchain, one needs a provider for that blockchain.
For now, a single generic `EIP155BlockchainProvider` is available in the `@iqprotocol/eip155` package:
it supports all EVM compatible blockchains. The SDK is extendable, and other providers can be created in the future.

Let's install dependencies to fetch some info and do some staking, renting and swapping:

```
npm i @iqprotocol/eip155  ethers typescript ts-node @types/node
```

and import the provider:

```typescript
import { EIP155BlockchainProvider } from '@iqprotocol/eip155';
```

Preparing a signer
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
that its RPC endpoint is `https://data-seed-prebsc-1-s1.binance.org:8545/` (or run our own node and use its endpoint),
take some valid network address (one of ours or just from an explorer), and
create a `VoidSigner` (this signer is suitable for "read-only" methods):

```typescript
import { VoidSigner, providers } from 'ethers';

const provider = new providers.JsonRpcProvider('https://data-seed-prebsc-1-s1.binance.org:8545/');
const clientAddress = '0x049153b8DAe0a232Ac90D20C78f1a5D1dE7B7dc5';
const signer = new VoidSigner(clientAddress, provider);
```

To check the connection, we can try to get the chain id (should be 97 in this case):

```typescript
blockchain.getChainId().then(({ reference }) => console.log(`chain id is ${reference}`));
```

For a "write" method (like swap, stake, or rent), another type of signer is required, for instance a Wallet:

```typescript
import { Wallet, providers } from 'ethers';

const provider = new providers.JsonRpcProvider('https://data-seed-prebsc-1-s1.binance.org:8545/');
const clientPrivateKey = '0xf3921efdfe42ea58356da0ab453ac073b6ab7a4a58f20aebfa408bbd57a91ee8';
const signer = new Wallet(clientPrivateKey, provider);
```

Learning about enterprises and services
---------------------------------------
Now that we have a signer and can connect to the network, we can also interact with an enterprise.

First, let's get some info about it. For example, for Parsiq enterprise in BSC Testnet we can run
the following code to get its metadata, services' addresses and metadata:

```typescript
(async() => {
    const enterpriseAddress = '0x9056eEF73095c76BfF1a5e0cEF14f99540643C72';

    const enterpriseInfo = await blockchain.enterprise(enterpriseAddress).getInfo();
    console.log(enterpriseInfo);

    const servicesAddresses = await blockchain.enterprise(enterpriseAddress).getServiceAddresses();
    console.log(servicesAddresses);

    for(const address of servicesAddresses) {
        console.log((await blockchain.service(address).getInfo()));
    }
})();
```

The "write" methods are called in a similar fashion.

Stake, rend and swap
--------------------

With a wallet signer, we can do various things with client's funds.
Usually all the operations are done with the enterprise's tokens, so let's assume you
have some already. We'll see that it's not the only option in some cases, though.

To make explanations simpler, let's call enterprise tokens "ENT".

### Staking

Staking means providing client's ENT tokens to the enterprise's pool to get a reward later.
The staking reward is added to the client's stake and is proportional to their share in the pool.
The rewards originate from fees to the enterprise, so they also depend on how popular are
the services of the enterprise.

There's a number of `Enterprise` methods to control staking. Let's stake some ENT first:

```typescript
const amount = 100;
const tx = await blockchain.enterprise(enterpriseAddress).stake(amount);
```

As a result, we get a stake token (an sNFT) which holds the staked amount and other info. Once we get it, we can call
`Enterprise.getStakeTokenIds` to get the id of the stake token(s). To find out the stake amount, we can use
`Enterprise.getStake(tokenId)`. There's also `increaseStake`, `decreaseStake` and `unstake` to manipulate the amount,
several metadata getters. Finally, there's `getStakingReward` and `claimStakingReward`.

Note: although creating more than one stake is technically possible, there's probably no reason to do so,
and this is generally discouraged.

Now let's consider two ways to get services' tokens: renting and swapping.

### Swapping

Swapping is a straightforward way of getting service tokens: pay some ENT tokens,
get some tokens of that service (also known as "power" tokens, let's call them "PWR"
since we only consider using one service at a time here). IQ SDK makes it as simple as it sounds:

```typescript
const swapInTx = await blockchain.service(serviceAddress).swapIn(amount);
```

An `amount` of ENT here is swapped to the same amount of PWR.
Likewise, they can be swapped back:

```typescript
const swapOutTx = await blockchain.service(serviceAddress).swapOut(amount);
```

Note that swapping can be forbidden for some services though.
This may depend on their functionality and financial model.
To figure out whether swapping is enabled, use

```typescript
const isSwappingEnabled = await blockchain.service(serviceAddress).isSwappingEnabled();
```

Similarly, transferring PWR to another client may be forbidden,
but transferring is out of the scope of this article.

### Renting

Swapping is a method of `Service` since it doesn't involve any fees in ENT (only tx gas fees);
renting, however, involves paying a fee to the enterprise, so it's a method of `Enterprise`.

If we want to rent `rentalAmount` of PWR of `serviceAddress` for `rentalPeriod`, we have to
pay a fee in some tokens identified by `paymentTokenAddress`. In most cases, the fee is paid
in ENT, but that's not obligatory. Since the fee depends on all these parameters,
it's reasonable to ask for an estimation first:

```typescript
const esitmatedFee = await blockchain.enterprise(enterpriseAddress)
    .estimateRentalFee(serviceAddress, paymentTokenAddress, rentalAmount, rentalPeriod);
```

Once we've come up with an amount that we are ok to pay, we can call

```typescript
const rentTx = await blockchain.enterprise(enterpriseAddress)
    .rent(serviceAddress, paymentTokenAddress, rentalAmount, rentalPeriod, maxPayment);
```

Note: there's also `Service.estimateRentalFee` which promises an object containing
three parts of the fee: pool, service and garbage collector fee. The latter may deservie a closer consideration.
In fact, on renting PWR, the client also gets another NFT that holds info about the rent.
Once rental period ends, the rented PWR no longer "work" and should be transfered back to the pool.
If the client does it themselves, they also get the garbage collector fee back.
On the other hand, after a certain period that operation becomes available for others
which creates a "crowd-sourced garbage collecting".

Statuses and gas fees
---------------------

Most methods return a promise of a `ContractTransaction` if there's no error.
Since it has a `hash` property, the status can be further fetched or watched via standart tools like web3
(for instance, using `web3.eth.getTransaction(hash)`).

Likewise, one can find out the spent gas by transaction hash.

Estimating gas fee is not currently provided by IQ SDK, but usually that's not necessary
as it is previewed by Metamask and can be tweaked anyway.

Wrapping up
-----------

We have discussed how to do basic operations with DeFi enterprises via IQ SDK: staking, renting and swapping.
___something about "much more"?