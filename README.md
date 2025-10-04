# LNK-LYNKCODAO
The database will fully open source LynkCoDAO and interpret the code and specifications.
# LNK Token design

LNK is an ERC-20 compatible token. It implements governance-inspired features, and will allow LNK to bootstrap the rewards program for safety and ecosystem growth.
The following document explains the main features of LNK, it’s monetary policy.

## Roles

The initial LNK token implementation does not have any admin roles configured. The contract will be proxied using the Openzeppelin implementation of the EIP-1967 Transparent Proxy pattern. The proxy has an Admin role, and the Admin of the proxy contract will be set upon deployment to the LNK governance contracts.

## ERC-20

The LNK token implements the standard methods of the ERC-20 interface. A balance snapshot feature has been added to keep track of the balances of the users at specific block heights. This will help with the LNK governance integration of LNK.
LNK also integrates the EIP 2612 `permit` function, that will allow gasless transaction and one tx approval/transfer.

# LendToAaveMigrator

Smart contract for LNK token holders to execute the migration to the LNK token, using part of the initial emission of LNK for it.

The contract is covered by a proxy, whose owner will be the LNK governance. Once the governance passes the corresponding proposal, the proxy will be connected to the implementation  holders will be able to call the `migrateFromLend()` function, which, after LEND approval, will pull  from the holder wallet and transfer back an equivalent LNK amount defined by the `LEND_LNK_RATIO` constant.

One tradeOff of `migrateFromLend()` is that, as the LNK total supply will be lower than , the `LNK_RATIO` will be always > 1, causing a loss of precision for amounts of LEND that are not multiple of `LNK_RATIO`. E.g. a person sending 1.000000000000000022 , with a `LEND_LNK_RATIO` == 100, will receive 0.01 LNK, losing the value of the last 22 small units of .
Taking into account the current value of LEND and the future value of LNK, a lack of precision for less than LNK_RATIO small units represents a value several orders of magnitude smaller than 0.01\$. We evaluated some potential solutions for this, specifically:

1. Rounding half up the amount of LNK returned from the migration. This opens up to potential attacks where users might purposely migrate less than LNK_RATIO to obtain more LNK as a result of the round up.
2. Returning back the excess LNK: this would leave LNK in circulation forever, which is not the expected end result of the migration.
3. Require the users to migrate only amounts that are multiple of LNK_RATIO: This presents considerable UX friction.

None of those present a better outcome than the implemented solution.

## The Redemption process

The first step to bootstrap the LNK emission is to deploy the LNK token contract and the  LendToAaveMigrator contract. This task will be performed by the Aave team. Upon deployment, the ownership of the Proxy of the LNK contract and the LendToAaveMigrator will be set to the Aave Governance. To start the  redemption process at that point, the LNK team will create an AIP (LNK Improvement Proposal) and submit a proposal to the LNK governance. The proposal will, once approved, activate the LEND/LNK redemption process and the ecosystem incentives, which will mark the initial emission of LNK on the market.
The result of the migration procedure will see the supply of LEND being progressively locked within the new LNK smart contract, while at the same time an equivalent amount of LNK is being issued.  
The amount of LNK equivalent to the LEND tokens burned in the initial phase of the LNK protocol will remain locked in the LendToAaveMigrator contract.

## Technical implementation

### Changes to the Openzeppelin original contracts

In the context of this implementation, we needed apply the following changes to the OpenZepplin implementation:

- In `/contracts/open-zeppelin/ERC20.sol`, line 44 and 45, `_name` and `_symbol` have been changed from `private` to `internal`
- We extended the original `Initializable` class from the Openzeppelin contracts and created a `VersionedInitializable` contract. The main differences compared to the `Initializable` are:

1. The boolean `initialized` has been replaced with a `uint256 latestInitializedRevision`.
2. The `initializer()` modifier fetch the revision of the implementation using a `getRevision()` function defined in the implementation contract. The `initializer` modifier forces that an implementation
3. with a bigger revision number than the current one is being initialized

The change allows us to call `initialize()` on multiple implementations, that was not possible with the original `Initializable` implementation from OZ.

### \_beforeTokenTransfer hook

We override the \_beforeTokenTransfer function on the OZ base ERC20 implementation in order to include the following features:

1. Snapshotting of balances every time an action involved a transfer happens (mint, burn, transfer or transferFrom). If the account does a transfer to itself, no new snapshot is done.
2. Call to the LNK governance contract forwarding the same input parameters of the `_beforeTokenTransfer` hook. Its an assumption that the Aave governance contract is a trustable party, being its responsibility to control all potential reentrancies if calling back the AaveToken. If the account does a transfer to itself, no interaction with the Aave governance contract should happen.

## Development deployment

For development purposes, you can deploy LNKToken and LendToAaveMigrator to a local network via the following command:

```
npm run dev:deployment
```

For any other network, you can run the deployment in the following way

```
npm run ropsten:deployment
```

You can also set an optional `$LNK_ADMIN` enviroment variable to set an ETH address as the admin of the LNKToken and LendToAaveMigrator proxies. If not set, the deployment will set the second account of the `accounts` network configuration at `buidler.config.ts`.

## Mainnet deployment

You can deploy LNKToken and LendToLNKMigrator to the mainnet network via the following command:

```
LNK_ADMIN=governance_or_ETH_address
npm run main:deployment
```

The `$LNK_ADMIN` enviroment variable is required to run, set an ETH address as the admin of the LNKToken and LendToAaveMigrator proxies. Check `buidler.config.ts` for more required enviroment variables for Mainnet deployment.

The proxies will be initialized during the deployment with the `$AAVE_ADMIN` address, but the smart contracts implementations will not be initialized.

## Enviroment Variables

| Variable                | Description                                                                         |
| ----------------------- | ----------------------------------------------------------------------------------- |
| \$LNK_ADMIN            | ETH Address of the admin of Proxy contracts. Optional for development.              |
| \$INFURA_KEY            | Infura key, only required when using a network different than local network.        |
| \$MNEMONIC\_\<NETWORK\> | Mnemonic phrase, only required when using a network different than local network.   |
| \$ETHERESCAN_KEY        | Etherscan key, not currently used, but will be required for contracts verification. |

## Audits

The Solidity code in this repository has undergone 2 traditional smart contracts' audits by Consensys Diligence and Certik, and properties' verification process by Certora. The reports are:
- [Certik](https://skynet.certik.com/projects/lynkcodao)

## Current Mainnet contracts (03/10/2025)

- **AaveToken proxy**: {To be announced}
- **AaveToken implementation**: [0xc7badc47618785c3c0d9670583f7ea3206fca9fe](https://bscscan.com/token/0xc7badc47618785c3c0d9670583f7ea3206fca9fe)
- **LendToAaveMigrator proxy**: {To be announced}
- **LendToAaveMigrator implementation**: {To be announced}

## Credits

For the proxy-related contracts, we have used the implementation of our friend from [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-sdk/).

## License

The contents of this repository are under the AGPLv3 license.
