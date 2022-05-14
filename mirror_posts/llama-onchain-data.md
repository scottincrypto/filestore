
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
