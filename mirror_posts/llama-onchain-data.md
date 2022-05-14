
# Onchain Data at Llama

Llama helps DAOs manage their treasuries and this entails a lot of reporting work.  For good reasons, the weapon of choice for this reporting is [Dune Analytics](https://dune.com).  Dune provides a great environment for doing on-chain analytics work.  There is an easy-to-learn graphical frontend for making dashboards, with Dune taking care of hosting and all the other backend plumbing.  The queries are written in SQL, which has been around forever and many people already know.  Dune decodes smart contract data so that it's findable via human readable table names, instead of the binary mess that are the internal blockchain data structures. 

## Building a Dune Dashboard

If you want to build a Dune dashboard, you need a few things first:

### You need to know SQL
This is the easy part.  If you know selects, joins & window functions then you are already 90% there.  If you don't know this, a couple of online courses will get you up to speed quickly.

### You need to know how the protocol works
A crypto protocol typically has a collection of smart contracts which execute on the blockchain.  When these contracts have their functions called, or they do somethign and emit an event, a record is written to the chain which gets picked up by Dune.  What these calls and events do, and what the parameters they record mean is the puzzle.  Some protocols have great documentation, others less so.  Often documentation is not up to date with the latest deployed contracts. Seemingly simple things, like calculating APR/APY, or how fees are accounted for are unique to each protocol.  Be prepared to comb through Github and read the deployed contract code on Etherscan to work out how things actually work.

### You need the contracts in question decoded on Dune
Smart contracts are compiled into bytecode which is executed on the blockchain.  This bytecode isn't human readable, nor is the data which they write to the chain.  Thankfully Dune takes care of this for us.  Using the ABI of a smart contract as the key, this bytecode can be translated back into human readable function calls and event data.  Dune handles this via a [contract decoding process](https://docs.dune.com/data-tables/data-tables/decoded-data). This requires a protocol to have published their contract ABIs publicly, either by verifying the contract on Etherscan or publishing a copy elsewhere.  It's surprising that this isn't always the case.  

If you're lucky, the protocol you want to make a dashboard for already has the contracts decoded.  If you're really lucky, they'll even be decoded correctly.  Decoding requests are submitted by users so the input data is not always reliable, and hence there can be problems with some contracts.  Thankfully, Dune have some friendly and helpful people to help out with decoding issues. 


## Building for TribeDAO - Putting it into Practice

As part of Llama's work with TribeDAO I built [this dashboard](https://dune.com/llama/Fei-Protocol-Lending-Markets) to track the lending rates for the FEI stablecoin across Aave, Compound and Rari Fuse.  This dashboard needed to calculate the amount of deposited FEI in each of these platform pools, as well as the APY received by depositors for both FEI and competing stablecoins.  Fortunately, Rari Fuse is a fork of Compound (with each pool a  separate instance of Compound) so the code and the accounting was very similar for both protocols.  Compound-like protocols 

In these Compound-like protocols, a user deposits FEI into a Pool and they receive an amount of cTokens in return.  These cTokens act as a fungible deposit receipts, representing a redeemable claim on the underlying assets.  As interest is earned on the deposited tokens, the exchange rate of the token/cToken increases (calculated by the smart contract) and a user can redeem their cTokens for an increased number of underlying assets at a later date.

### An Easy Problem
Calculating total deposits for FEI (or any token) on a Compound-like protocol is simple in Dune.  The deposit and withdrawal transactions emit a Mint event (deposit FEI, mint cFEI) or a Redeem event (withdraw FEI, burn cFEI) which contain the quanties of the cToken and underlying FEI in the transaction.  Here is an event from Etherscan illustrating this for the Rari Fuse Pool 8 FEI deposit contract:

[![Fuse Pool 8 FEI Redeem Event](https://github.com/scottincrypto/filestore/raw/main/mirror_posts/fuse_pool_8_redeem_event.png)](https://etherscan.io/address/0xd8553552f8868C1Ef160eEdf031cF0BCf9686945#events)

### A Harder Problem

Calculating deposit APR is a bit more challenging.  The APY is calculated internally within the smart contract on a block by block basis.  This can be seen via etherscan - for Fuse Pool 8 FEI, go to [the cToken Contract](https://etherscan.io/address/0xd8553552f8868C1Ef160eEdf031cF0BCf9686945#readProxyContract) and expand the "SupplyRatePerBlock" function.  Divide this by 10^18 and you have the interest rate for the current block.  Multiply this by the number of blocks per year and you get an annualised rate.

Dune doesn't let us read contracts directly like this.  We have to rely on the events which only occur when a transaction is written to the chain, such as the Mint and Redeem events above.  With this limitation we can only infer the APR via this method:
- Take one event and calculate the exchange rate
- Take another event and calculate the exchange rate
- Divide the change in exchange rate by the elapsed time between events to get the approximate return for that time period
- Multiply out to get the annualised rate

This approach works OK when there are lots of transactions - you get a continous series of snapshots of the exchange rate to calculate the APR over your target time period.  A large number of transactions means your inferred value will approach that of the internal smart contract calculation (the correct answer).  What happens, however, if there are no transactions in the time period you are interested?  It is impossible to calculate the APR from the on-chain events, even though the contract has one calculated internally.  Worse still, if there are only a small number of transactions then the inferred APR may be a long way from the internal smart contract value.  This can be seen on the above dashboard for Fuse Pools beyond Pool 18, where liquidity is thin and transactions are sporadic.

### Solving the impossible

There are other problems which simply can't be solved using Dune alone.  When building the Fuse Pool portion of the dashboard above, it was important to know:
- The names & numbers & contracts of each of the Fuse Pools (there are nearly 200)
- The assets which are in each pool, including matching the cToken address to the underlying asset address.  There are nearly 700 Fuse assets across all the pools.

Dune only had 200 assets showing in the tables, and there was no table available which mapped Fuse Pools, cTokens and underlying assets.  Further investigation revealed that cTokens are created in at least three different ways, with some being via multicall operations (where call data is obscured) and via unverified contracts.  It was impossible to get the full picture of Fuse assets via Dune.

Like the APR problem, this can be easily solved by directly querying the Fuse smart contracts.
- The FusePoolDirectory contract (https://etherscan.io/address/0x835482FE0532f169024d5E9410199369aAD5C77E) has a method called getAllPools() which returns a list of pool names and contracts. 
- Each pool contract has a method called getAllMarkets(), which returns the list of cTokens in a pool.  
- Each cToken contract has a method called underlying() which returns the contract address of the underlying asset.  

Dune allows users to submit code which creates custom tables or views, known as abstractions.  These can be used to cache data for faster access, or add external data to Dune for access in queries.

To get the Fuse contract data into Dune, I wrote code to iterate through the Fuse smart contracts and generate an SQL table creation query for an abstraction. I used the python library eth-brownie <link> for this, although it can be done in web3.py or equivalent libraries in other languages.  The code can be found [here](https://github.com/scottincrypto/dune-rari-fuse-assets) and the resulting abstraction is available in Dune as [rari_capital.view_rari_fuse_ftokens.](https://github.com/duneanalytics/abstractions/tree/master/ethereum/rari_capital)

![Dune Fuse Pools](https://github.com/scottincrypto/filestore/raw/main/mirror_posts/dune_fuse_pools.png)

## Llamas Delivering

As Llama prepared the [Quarterly Financial Report for TribeDAO](https://llama.xyz/reports/tribe/q1-presentation.pdf), it became apparent that some of the data was going to be very difficult or even impossible to get via Dune reports.  After the FEI-Rari merger, Rari income needed to be included in the financial reports.  Income from Fuse Pools is generated for TribeDAO via the Platform Fee where a percentage (usually 10% but not always) of the interest charged on borrowing is kept as a fee.  This fee isn't reported out via any events from the Fuse contracts and the fees accrue inside the Fuse contracts for retrieval by the Rari team when required.  Thus there are no token transfer events which can reliably account for the Fuse fees earned by the protocol.  This is a problem which simply can't be solved in Dune, or any other on-chain analytics platform which uses events & calls (Flipside, Covalent etc).  

At Llama, we like to test the boundaries of impossible.  Our Fuse pool income problem, unsolvable on Dune, can be attacked by directly querying the contracts.  Each Fuse cToken contract has a totalFuseFees() method which returns the fees accrued in the contract.  By querying an archive node using python with eth-brownie, the fees accrued over time for each contract could be calculated then aggregated into a Fuse income statement.  

Llama is building out a suite of internal tools like this to accurately and efficiently report on the financial and protocol performance of our clients.  Tools like Dune make sense in a lot of applications but it's important to understand the limitations.  Llama will use a range of technologies to solve these problems, and build the tools required if they aren't already available.
