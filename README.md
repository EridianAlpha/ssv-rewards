# ssv-rewards

Synchronizes historical activity and performance of SSV validators and calculates their rewards according to [Incentivized Mainnet Program](https://docs.google.com/document/d/1pcr8QVcq9eZfiOJGrm5OsE9JAqdQy1F8Svv1xgecjNY).

## Installation

```bash
git clone https://github.com/bloxapp/ssv-rewards
cd ssv-rewards
cp .env.example .env
cp rewards.example.yaml rewards.yaml
```

Edit `.env` and fill in the required values:

```ini
# SSV network
NETWORK=mainnet

# Beacon API endpoint of the consensus node
CONSENSUS_ENDPOINT=http://beacon-node:5052

# JSON-RPC API endpoint of the execution node
EXECUTION_ENDPOINT=http://excution-node:8545

# SSV API endpoint
SSV_API_ENDPOINT=https://api.ssv.network/api/v4

# Beaconcha.in API
BEACONCHA_ENDPOINT=https://beaconcha.in
BEACONCHA_API_KEY= # Optional
BEACONCHA_REQUESTS_PER_MINUTE=20 # Adjust according to your Beaconcha.in API plan

# Etherscan API
ETHERSCAN_API_ENDPOINT=https://api.etherscan.io
ETHERSCAN_API_KEY= # Optional
ETHERSCAN_REQUESTS_PER_SECOND=0.1 # Adjust according to your Etherscan API plan
```

Edit `rewards.yaml` to match [the specifications](https://docs.google.com/document/d/1pcr8QVcq9eZfiOJGrm5OsE9JAqdQy1F8Svv1xgecjNY):

```yaml
criteria:
  min_attestations_per_day: 202
  min_decideds_per_day: 22

tiers:
  # Tiers apply to rounds below the participation threshold.
  - max_participants: 2000 # Up to 2,000 validators
    apr_boost: 0.5 # Fraction of ETH APR to reward in SSV tokens
  # ...
  - max_participants: ~ # Limitless
    apr_boost: 0.1

rounds:
  - period: 2023-07 # Designated period (year-month)
    eth_apr: 0.047 # ETH Staking APR
    ssv_eth: 0.0088235294 # SSV/ETH price
  # ...
```

## Usage

First, start PostgreSQL and wait a few seconds for it to be ready:

```bash
docker-compose up -d postgres
```

### Synchronization

Synchronize validator activity and performance:

```bash
docker-compose run --rm sync
```

_This might take a while, depending on how long ago the SSV contract was deployed and how many validators there are._

### Calculation

After syncing, you may calculate the reward distribution:

```bash
docker-compose run --rm calc
```

This produces the following documents under the `./rewards` directory:

```bash
📂 rewards
├── 📄 by-owner.csv            # Reward per round for each owner
├── 📄 by-validator.csv        # Reward per round for each validator
├── 📄 by-recipient.csv        # Reward per round for each recipient
├── 📄 total-by-owner.csv      # Total reward for each owner
├── 📄 total-by-validator.csv  # Total reward for each validator
└── 📂 <year>-<month>
    ├── 📄 by-owner.csv        # Total reward for each owner for this month
    ├── 📄 by-validator.csv    # Total reward for each validator for this month
    ├── 📄 by-recipient.csv    # Total reward for each recipient for this month
    └── 📄 cumulative.json     # Cumulative reward for each owner until and including this month
```

- `recipient` is the address that eventually receives the reward, which is either the owner address, or if the owner is a contract, then the deployer address of the contract.

### Merkleization

After calculating the reward distribution, you may merkleize the rewards for a specific round.

1. Copy the file at `./rewards/<year>-<month>/cumulative.json` over to `./scripts/merkle-generator/scripts/input-1.json`.
2. Run the merkleization script:
   ```bash
   cd merkle-generator
   npm i
   npx hardhat run scripts/merkle.ts
   ```
3. The merkle tree is generated at `./merkle-generator/scripts/output-1.json`.
