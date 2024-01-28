Finally It's time to talk about the final project and as you already know you will have to implement NFT Marketplace on Solana using Anchor framework.

### introduction
For introduction let's just forget about implementation part and go through all the functionalities typical NFT marketplace provides. To be honest there is not a lot of complicated details about the process of listing and buying nfts on a marketplace. Lets just go through all the available operations:

* **List an Nft** - to list an NFT, user just provides the price and the nft address and presses **List** button. UI might vary for different marketplaces but main concept is the same.
* **Update Price**  - User who listed an NFT should be able to update the price of this nft
* **Cancel Listing** - Users should be able to Cancel nft listings if they decide not to sell their NFTs
* **Buy an NFT** - Of course there should also be possibility to buy listed NFT for somebody else.

That's basic functionality what your marketplace should implement, I guess it doesn't seem so hard. But there are additional functionality our marketplace needs for basic needs. 
Fist of all user needs to see all NFTs that are listed on a marketplace, otherwise they won't be able to choose which NFTs to buy. As we know we won't be using traditional web2 backend for this assignment, so you will have to query blockchain for this kind of data. 
Also user who wants to list his nfts should have a way to choose an nft for listing, he should not be required to copy and paste NFT mint address - that would be terrible User experience. 
We will talk more about this querying stuff later. Not let's discuss some technical details!

### Technical Details and Requirements
#### Anchor
You will be using anchor framework to develop a smart contract. preferably `v0.29.0`.
You will also have to implement client part with typescript, similarly to the **Splitwise Assignment**. In this part you will also be using Anchor Framework's typescript library, you can install it like so:
```shell
yarn add @coral-xyz/anchor
```

#### Project Structure
In this project I decided to make some stub classes to make it easier to see the full picture. in `src/lib.rs` file there will be stub methods for all the instructions your smart contract should support, there will also be `#[derice(Accounts)]` structs for all of them but you should fill them up with needed accounts.
I will also provide account structs to store the program state in, some fields will be present and some fields will be added by you.

#### Accounts

##### Global State
First of all there will be a `GlobalState` account, that will store meta information about our marketpalce instance, each `GlobalState` will be associated with one marketpalce instance, so users will have an ability to create multiple marketpalce instances using smart contract deployed by you.
```rust
pub struct GlobalState {

	pub initializer: Pubkey,
	pub total_listed_count_sol: u32,
	pub total_listed_count_spl: u32,

	pub total_volume_all_time_sol: u128,

	pub all_time_sale_count_spl: u64,
	pub all_time_sale_count_sol: u64,
	pub marketplace_fee_percentage: u64,
}
```
as you can see there are already some fields here, let's go through them:
* `initializer` - account who initialized this global state and basically owns a marketplace, you can send marketplace fees from sales to this account or add one specifically for that purpose. Also note that address of the Global state should be derived from `initializer`, and also one user should be able to create multiple global state accounts - you should come up with an algorithm of how to achieve this yourself.
* `total_listed_count_sol` and `total_listed_count_spl` should store and dynamically update the count of listed nfts - both in Solana and in SPL tokens. Yes you heard it right, users should be able to list NFTs in their preferred SPL tokens - for example list nft in `USDC` or `BONK`. we will discuss more about this later. Of course when someone unlists their NFT these values should also decrease.
* `all_time_sale_count_spl` and `all_time_sale_count_sol` - total number of nft purchases that happend both in SOL and in all other SPL tokens combined.
* `total_volume_all_time_sol` - total amount of `SOL` across all marketplace trades. you should sum SOL amounts including **creator** and **marketplace** fees.
* `marketplace_fee_percentage` - it's config value specifying what percentage of a sale amount should go to the owner of the marketplace. 

##### Listing Account 
```rust
pub struct Listing {
	// Marketplace instance global state address
	pub global_state_address: Pubkey,
	// User who listed this nft
	pub initializer: Pubkey,
	// NFT mint address
	pub nft_mint_address: Pubkey,
	// Program PDA account address, who holds NFT now
	pub nft_holder_address: Pubkey,
	// Price of this NFT.
	pub price: u64,
	// listing creation time
	pub creation_time: i64,
	pub updated_at: i64,
	// if trade payment is in spl token currency
	pub is_spl_listing: bool,
	// trade spl token address
	pub trade_spl_token_mint_address: Pubkey,
	pub trade_spl_token_seller_account_address: Pubkey,
}
```
`Listing` will be created after user lists an nft on a marketplace. each listed nft will cause the creation of this type of account. Address of this account should be derived from `global_state_address`, `listing initializer address` and an `nft address`, so that several people can list multiple NFTs on the same or different marketplaces. It seems like Listing account has whole lot of fields so let's go through each of them:
* `global_state_address` - address of the global state account this listing is associated with
* `initializer` - listing initializer account, after a sale this account should receive an NFT price
* `nft_mint_address` - I hope you can guess what will be stored here
* `nft_holder_address` - As you can remember nft is an SPL token with total supply of `1`, so program somehow should be able to get a hold of this Token when the listing is created. This address should be a program derived address which will be the temporary owner of the NFT while the listing is active. You can derive this address from listing account address, or you can have one global nft holder address which will own all the listed NFT-s, that's how Magic Eden does it: [Transaction](https://solscan.io/tx/5qPyjHdUCNoMr7gv4ZXn8Qq8xjHbGopSDscudrbiKPfUg67YL8jEiifMspb5kv6Gsy9AxDwvaQC2G2p3wM1vxLG7)
* `price` - price of an nft in lamports
* `is_spl_listing` - this boolean field should indicate wether NFT is listed in some SPL token or SOL
* `trade_spl_token_mint_address` - if NFT is listed in SPL this field should store tokens mint address
* `trade_spl_token_seller_account_address` - this field should indicate token account address of the seller, we need this to send the seller spl tokens when the someone buys an NFT.

The fields specified here are not mandatory they are just one way of implementation - you can add or remove any fields to align your own implementation if it's more comfortable.

#### Instructions

##### Initialize marketplace
`InitializeMarketplace` instruction should create new `GlobalState` account with all required data. Don't forget to make it PDA of the program. `marketplace_fee_precentage` should be passed as a parameter as well. Also users should be able to create multiple global states so take in into consideration.

##### List Nft
`ListNft` instruction should create Listing Account and transfer NFT to some Holder address associated with this listing account, User Also should pass the price of the NFT which will be set in `price` field of the listing account. Listing initializer should pay for the Rent of this account. Also the Rent of the token account which will store the NFT.

##### List NFT in SPL token
For listing in spl tokens you can have different instruction, as some things might be different for Transfering `SOL` and this approach gives us less branching in code. Additionaly to the fields from regural listing instruction this one should also Accept SPL token mint address in which initializer wants to list his or her NFT. As well as his/her token account address for this NFT.

##### Cancel Listing
Listing initializer should also be able to Close Listing and Holder token accounts, Listed NFT and `SOL` amount he payed for rent should be returned to the initializer. Of course you must make sure that the account is only possible to be closed by the initializer.

##### Update Price
Initializer should also be able to update price of the listed NFT, without closing the listing account. 

##### Buy NFT
Sender of this instruction should receive listed nft and the price amount should be transferred to the initializer. Also take into consideration that marketplace should also receive a percentage of the sell amount and creators of the nft should also receive their fare share. We will discuss this process in more detail later.

##### Buy NFT in SPL
The same functionality as Buy NFT, but transactions take place in SPL token.

#### Example Program
There's already a similar program deployed on mainnet: you can see it's [transactions](https://solscan.io/account/657iw8S9b4BG5Vno91DgJk4bqoH3kzPRopngPG8uxWxg) to inspect how it works.

#### Buy NFT Transaction overview
Here's a purchase [transaction](https://solscan.io/tx/2qC2mcMJgsMYYzmLwBBWKo7TRcKcxR2UFQNejVuxDRe6RBqbxDc81y629F16zVWVYRPnDDZcdByTdEiDUKgub6e1) of the marketplace. Nft was sold for 0.95 `SOL`. You can see that buyer `F5UMZd9kW9bJZnyaZpnsFpVrm2hwETFLUf42636zhy1t` transfers this amount to the 3 different accounts. The seller receives highest chunk of the amount, 1% of the sell amount goes to the marketplace as a fee and 5% goes to the NFT creator. List of NFT creators and their share are stored in NFT Metadata `Creators` field. `SellerFeeBasisPoints` - specifies what fraction of the amount should creators get. here 100 points equals 1%. so 500 will be 5%. Then each creator has hist designated value of how much % of this basis points he would receive.
For example [here](https://solscan.io/token/CdVCPRpUDP8yxfMqpmAb3qBaasbLjbqHhaYkr6HBdoSF#metadata) `sellerFeeBasisPoints` = 420, so creators would receive 4.2% of the sale. and in creators list there are two accounts:
```json
[
  {
    "address": "5XvhfmRjwXkGp3jHGmaKpqeerNYjkuZZBYLVQYdeVcRv",
    "verified": 1,
    "share": 0
  },
  {
    "address": "2RtGg6fsFiiF1EQzHqbd66AhW7R5bWeQGpTbv2UMkCdW",
    "verified": 1,
    "share": 100
  }
]
```
as you can see `2RtGg6fsFiiF1EQzHqbd66AhW7R5bWeQGpTbv2UMkCdW` will receive 100% of the share.

you should account for the creators in your marketplace and transfer them fractions of the sell amount. using anchor's `remaining_accounts` might come in handy when handling creators.

#### Some security checks you should make
* there should be minimum and maximum price for NFT listing
* Users should not be able to list regular tokens, you should make sure that real NFT is being listed.
* after Listing is cancelled or NFT is purchased you should `close` all accounts you created for listing and return `SOL` to the listing initializer.

#### The Client

Client is pretty straight forward, you can find client method signatures in `client/marketplace-client.ts` file. there are 7 methods for implementing sending specific instructions, but you have some restrictions on those functions:
* you shouldn't use anchor's `.rpc()` method to call the methods directly, you can only generate instructions with anchor framework, you can only use `@solana/web3.js` for other stuff. 
* you should also wait for transaction to confirm/finalize and only then return the function, if transaction fails or gets skipped you can throw an exception or retry. Transactions usually are considered as skipped if they aren't confirmed within 30 seconds.

#### data retrieval
there are also 4 data retrieval methods:
* `getMarketplaceMetadata` this just fetches globalState fields, you can set global state address in constructor.
* `getUserListings` fetches all nft listings with corresponding nft data made by specified account
* `getAllListings` fetches all listings on the marketplace
* `getUserNfts` fetches all the NFTs posessed by user. But this method has a small restriction, you can only use `@solana/web3.js` library, you can't use metaplex's `metaplex.nfts()` method to do it, because it would be too easy.

i will update this document and write more detailed description if it will be needed.