<div align="center">
  <img height="120x" src="https://i.ibb.co/6Dn2D1s/2r.png" />

  <h1 style="margin-top:20px;">LFG Protocol v2</h1>

  <p>
    
  </p>
</div>

## Installation

```
npm i @LFG-labs/sdk
```

## Getting Started

### Setting up a wallet for your program

```bash
# Generate a keypair
solana-keygen new

# Get the pubkey for the new wallet (You will need to send USDC to this address to Deposit into LFG (only on mainnet - devnet has a faucet for USDC))
solana address

# Put the private key into your .env to be used by your bot
cd {projectLocation}
echo BOT_PRIVATE_KEY=`cat ~/.config/solana/id.json` >> .env
```

## Concepts

### BN / Precision

The LFG SDK uses BigNum (BN), using [this package](https://github.com/indutny/bn.js/), to represent numerical values. This is because Solana tokens tend to use levels of precision which are too precise for standard Javascript floating point numbers to handle. All numbers in BN are represented as integers, and we will often denote the `precision` of the number so that it can be converted back down to a regular number.

```bash
Example:
a BigNum: 10,500,000, with precision 10^6, is equal to 10.5 because 10,500,000 / 10^6 = 10.5.
```

The LFG SDK uses some common precisions, which are available as constants to import from the SDK.

| Precision Name        | Value |
| --------------------- | ----- |
| FUNDING_RATE_BUFFER   | 10^3  |
| QUOTE_PRECISION       | 10^6  |
| PEG_PRECISION         | 10^6  |
| PRICE_PRECISION       | 10^6  |
| AMM_RESERVE_PRECISION | 10^9  |
| BASE_PRECISION        | 10^9  |

**Important Note for BigNum division**

Because BN only supports integers, you need to be conscious of the numbers you are using when dividing. BN will return the floor when using the regular division function; if you want to get the exact division, you need to add the modulus of the two numbers as well. There is a helper function `convertToNumber` in the SDK which will do this for you.

```typescript
import {convertToNumber} from @LFG-labs/sdk

// Gets the floor value
new BN(10500).div(new BN(1000)).toNumber(); // = 10

// Gets the exact value
new BN(10500).div(new BN(1000)).toNumber() + BN(10500).mod(new BN(1000)).toNumber(); // = 10.5

// Also gets the exact value
convertToNumber(new BN(10500), new BN(1000)); // = 10.5
```

## Examples

### Setting up an account and making a trade

```typescript
import { AnchorProvider, BN } from '@coral-xyz/anchor';
import { Token, TOKEN_PROGRAM_ID } from '@solana/spl-token';
import { Connection, Keypair, PublicKey } from '@solana/web3.js';
import {
	calculateReservePrice,
	LFGClient,
	User,
	initialize,
	PositionDirection,
	convertToNumber,
	calculateTradeSlippage,
	PRICE_PRECISION,
	QUOTE_PRECISION,
	Wallet,
	PerpMarkets,
	BASE_PRECISION,
	getMarketOrderParams,
	BulkAccountLoader,
	getMarketsAndOraclesForSubscription
} from '../sdk';

export const getTokenAddress = (
	mintAddress: string,
	userPubKey: string
): Promise<PublicKey> => {
	return Token.getAssociatedTokenAddress(
		new PublicKey(`ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL`),
		TOKEN_PROGRAM_ID,
		new PublicKey(mintAddress),
		new PublicKey(userPubKey)
	);
};

const main = async () => {
	const env = 'devnet';
	// Initialize LFG SDK
	const sdkConfig = initialize({ env });

	// Set up the Wallet and Provider
	const privateKey = process.env.BOT_PRIVATE_KEY; // stored as an array string
	const keypair = Keypair.fromSecretKey(
		Uint8Array.from(JSON.parse(privateKey))
	);
	const wallet = new Wallet(keypair);

	// Set up the Connection
	const rpcAddress = process.env.RPC_ADDRESS; // can use: https://api.devnet.solana.com for devnet; https://api.mainnet-beta.solana.com for mainnet;
	const connection = new Connection(rpcAddress);

	// Set up the Provider
	const provider = new AnchorProvider(
		connection,
		wallet,
		AnchorProvider.defaultOptions()
	);

	// Check SOL Balance
	const lamportsBalance = await connection.getBalance(wallet.publicKey);
	console.log('SOL balance:', lamportsBalance / 10 ** 9);

	// Misc. other things to set up
	const usdcTokenAddress = await getTokenAddress(
		sdkConfig.USDC_MINT_ADDRESS,
		wallet.publicKey.toString()
	);

	// Set up the LFG Client
	const LFGPublicKey = new PublicKey(sdkConfig.LFG_PROGRAM_ID);
	const bulkAccountLoader = new BulkAccountLoader(
		connection,
		'confirmed',
		1000
	);
	const LFGClient = new LFGClient({
		connection,
		wallet: provider.wallet,
		programID: LFGPublicKey,
		...getMarketsAndOraclesForSubscription(env),
		accountSubscription: {
			type: 'polling',
			accountLoader: bulkAccountLoader,
		},
	});
	await LFGClient.subscribe();

	// Set up user client
	const user = new User({
		LFGClient: LFGClient,
		userAccountPublicKey: await LFGClient.getUserAccountPublicKey(),
		accountSubscription: {
			type: 'polling',
			accountLoader: bulkAccountLoader,
		},
	});

	//// Check if user account exists for the current wallet
	const userAccountExists = await user.exists();

	if (!userAccountExists) {
		//// Create a Clearing House account by Depositing some USDC ($10,000 in this case)
		const depositAmount = new BN(10000).mul(QUOTE_PRECISION);
		await LFGClient.initializeUserAccountAndDepositCollateral(
			depositAmount,
			await getTokenAddress(
				usdcTokenAddress.toString(),
				wallet.publicKey.toString()
			)
		);
	}

	await user.subscribe();

	// Get current price
	const solMarketInfo = PerpMarkets[env].find(
		(market) => market.baseAssetSymbol === 'SOL'
	);

	const [bid, ask] = calculateBidAskPrice(
		LFGClient.getPerpMarketAccount(marketIndex).amm,
		LFGClient.getOracleDataForPerpMarket(marketIndex)
	);

	const formattedBidPrice = convertToNumber(bid, PRICE_PRECISION);
	const formattedAskPrice = convertToNumber(ask, PRICE_PRECISION);

	console.log(
		`Current amm bid and ask price are $${formattedBidPrice} and $${formattedAskPrice}`
	);

	// Estimate the slippage for a $5000 LONG trade
	const solMarketAccount = LFGClient.getPerpMarketAccount(
		solMarketInfo.marketIndex
	);

	const slippage = convertToNumber(
		calculateTradeSlippage(
			PositionDirection.LONG,
			new BN(1).mul(BASE_PRECISION),
			solMarketAccount,
			'base',
			LFGClient.getOracleDataForPerpMarket(solMarketInfo.marketIndex)
		)[0],
		PRICE_PRECISION
	);

	console.log(`Slippage for a 1 SOL-PERP would be $${slippage}`);

	await LFGClient.placePerpOrder(
		getMarketOrderParams({
			baseAssetAmount: new BN(1).mul(BASE_PRECISION),
			direction: PositionDirection.LONG,
			marketIndex: solMarketAccount.marketIndex,
		})
	);
	console.log(`Placed a 1 SOL-PERP LONG order`);
};

main();
```

## License

LFG Protocol v2 is licensed under [Apache 2.0](./LICENSE).

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in LFG SDK by you, as defined in the Apache-2.0 license, shall be
licensed as above, without any additional terms or conditions.

