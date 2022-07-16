# Add stake to validator

```
# in cli pod run 

call factory.shardnet.near create_staking_pool '{"staking_pool_id": "scholtz", "owner_id": "scholtz.shardnet.near", "stake_public_key": "ed25519:AeBRAFXbQcv9tYp2bNxBaoB6AsfC3utCJuAwaCWvNuXF", "reward_fee_fraction": {"numerator": 1, "denominator": 100}, "code_hash":"DD428g9eqLL8fWUxv8QSpVFzyHi1Qd16P8ephYCTmMSZ"}' --accountId="scholtz.shardnet.near" --amount=30 --gas=300000000000000

near call scholtz.factory.shardnet.near deposit_and_stake --amount 500 --accountId scholtz.shardnet.near --gas=300000000000000

near call scholtz.factory.shardnet.near ping '{}' --accountId scholtz.shardnet.near --gas=300000000000000

near view scholtz.factory.shardnet.near is_account_unstaked_balance_available '{"account_id": "scholtz.shardnet.near"}'

```
