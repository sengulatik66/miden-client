---
comments: true
---

To use the Miden client library in a Rust project, include it as a dependency. 

In your project's `Cargo.toml`, add:

```toml
miden-client = { version = "0.2" }
```

### Features

The Miden client library supports the [`testing`](https://github.com/0xPolygonMiden/miden-client/blob/main/docs/install-and-run.md#testing-feature) and [`concurrent`](https://github.com/0xPolygonMiden/miden-client/blob/main/docs/install-and-run.md#concurrent-feature) features which are both recommended for developing applications with the client. To use them, add the following to your project's `Cargo.toml`:

```toml
miden-client = { version = "0.2", features = ["testing", "concurrent"] }
```

## Client instantiation

Spin up a client using the following Rust code and supplying a store and RPC endpoint. 

The current supported store is the `SqliteDataStore`, which is a SQLite implementation of the `Store` trait.

```rust
let client: Client<TonicRpcClient, SqliteDataStore> = {
    
    let store = Store::new((&client_config).into()).map_err(ClientError::StoreError)?;

    Client::new(
        
        client_config,
        TonicRpcClient::new(&rpc_endpoint),
        SqliteDataStore::new(store),

    )?
};
```

## Create local account

With the Miden client, you can create and track any number of on-chain and local accounts. For local accounts, the state is tracked locally, and the rollup only keeps commitments to the data, which in turn guarantees privacy.

The `AccountTemplate` enum defines the type of account. The following code creates a new local account:

```rust
let account_template = AccountTemplate::BasicWallet {
    mutable_code: false,
    storage_mode: AccountStorageMode::Local,
};
    
let (new_account, account_seed) = client.new_account(account_template)?;
```
Once an account is created, it is kept locally and its state is automatically tracked by the client.

To create an on-chain account, you can specify `AccountStorageMode::OnChain` like so:

```Rust
let account_template = AccountTemplate::BasicWallet {
    mutable_code: false,
    storage_mode: AccountStorageMode::OnChain,
};

let (new_account, account_seed) = client.new_account(client_template)?;
```

The account's state is also tracked locally, but during sync the client updates the account state by querying the node for the most recent account data.

## Execute transaction

In order to execute a transaction, you first need to define which type of transaction is to be executed. This may be done with the `TransactionTemplate` enum. The `TransactionTemplate` must be converted into a `TransactionRequest`, which is a more general definition of a transaction.

Here is an example for a `pay-to-id` transaction type:

```rust
// Define asset
let faucet_id = AccountId::from_hex(faucet_id)?;
let fungible_asset = FungibleAsset::new(faucet_id, *amount)?.into();

let sender_account_id = AccountId::from_hex(bob_account_id)?;
let target_account_id = AccountId::from_hex(alice_account_id)?;
let payment_transaction = PaymentTransactionData::new(
    fungible_asset,
    sender_account_id,
    target_account_id,
);

let transaction_template: TransactionTemplate = TransactionTemplate::P2ID(payment_transaction);
let transaction_request = client.build_transaction_request(transaction_template).unwrap();

// Execute transaction. No information is tracked after this.
let transaction_execution_result = client.new_transaction(transaction_request.clone())?;

// Prove and submit the transaction, which is stored alongside created notes (if any)
client.send_transaction(transaction_execution_result).await?
```

You may also execute a transaction by manually defining a `TransactionRequest` instance. This allows you to run custom code, with custom note arguments as well.
