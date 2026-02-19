# miden-client-interactions

## requirements
- miden client setup
  here is the [guide](https://github.com/0xchukss/How-to-setup-a-miden-client)
- vsCode

# Step 1 create masm folder in client directory
run,
<pre>
  mkdir masm
  cd masm
</pre>

<img width="443" height="68" alt="mkdir masm" src="https://github.com/user-attachments/assets/6bb382fc-709b-4b7d-9f8f-c18204aabd81" />

## create scripts and accounts directory inside masm
run,
<pre>
  mkdir scripts
  mkdir accounts
</pre>

<img width="379" height="68" alt="mkdir scripts and accounts" src="https://github.com/user-attachments/assets/f4f02bab-5181-4f8c-80b0-351229dd669f" />

## create counter.masm file
navigate to accounts in masm folder 
run, 
<pre>
  cd accounts
</pre>

open this directory in vscode editor,
<pre>
  code .
</pre>

create a counter.masm file in this directory and paste this code;

<pre>
  use miden::protocol::active_account
use miden::protocol::native_account
use miden::core::word
use miden::core::sys

# Define a name for the storage "drawer" where we keep the number
const COUNTER_SLOT = word("miden::tutorials::counter")

#! Inputs:  []
#! Outputs: []
pub proc increment_count
    # 1. Get the current value from storage
    push.COUNTER_SLOT[0..2] exec.active_account::get_item
    
    # 2. Add 1 to the value
    add.1
    
    # 3. Put the new value back into storage
    push.COUNTER_SLOT[0..2] exec.native_account::set_item
    
    # 4. Clean up the stack
    exec.sys::truncate_stack
end
</pre>

<img width="683" height="428" alt="create counter masm" src="https://github.com/user-attachments/assets/ce1a8835-8a04-4a0a-a83f-6ac4a7be9c1b" />

# step 2 create faucet and counter contract
- navigate to src folder in main directory
- open the main.rs file
- replace the whole content with this,

<pre>
  use miden_client::auth::AuthFalcon512Rpo;
use rand::RngCore;
use std::sync::Arc;
use tokio::time::Duration;
use miden_client::{
    account::{
        component::{BasicFungibleFaucet, BasicWallet},
        AccountBuilder, AccountStorageMode, AccountType,
    },
    asset::{FungibleAsset, TokenSymbol},
    auth::AuthSecretKey,
    builder::ClientBuilder,
    keystore::FilesystemKeyStore,
    rpc::{Endpoint, GrpcClient},
    transaction::TransactionRequestBuilder,
    ClientError, Felt,
};
use miden_client_sqlite_store::ClientBuilderSqliteExt;

#[tokio::main]
async fn main() -> Result<(), ClientError> {
    // --- 1. INITIALIZE CLIENT ---
    // Fixed the port error by using Some(443)
    let endpoint = Endpoint::new(
        "https".to_string(), 
        "rpc.testnet.miden.io".to_string(), 
        Some(443)
    );
    
    let rpc_client = Arc::new(GrpcClient::new(&endpoint, 10_000));
    let keystore_path = std::path::PathBuf::from("./keystore");
    let keystore = Arc::new(FilesystemKeyStore::new(keystore_path).unwrap());
    let store_path = std::path::PathBuf::from("./store.sqlite3");

    let mut client = ClientBuilder::new()
        .rpc(rpc_client)
        .sqlite_store(store_path)
        .authenticator(keystore.clone())
        .build()
        .await?;

    println!("Client started! Syncing with Testnet...");
    client.sync_state().await?;
    println!("Successfully synced!");

    // --- 2. CREATE ALICE ---
    println!("\n[Step 1] Creating Alice's account...");
    let mut alice_seed = [0u8; 32];
    client.rng().fill_bytes(&mut alice_seed);
    let alice_key = AuthSecretKey::new_falcon512_rpo();

    let alice_account = AccountBuilder::new(alice_seed)
        .account_type(AccountType::RegularAccountUpdatableCode)
        .storage_mode(AccountStorageMode::Public)
        .with_auth_component(AuthFalcon512Rpo::new(alice_key.public_key().to_commitment()))
        .with_component(BasicWallet)
        .build()
        .unwrap();

    client.add_account(&alice_account, false).await?;
    keystore.add_key(&alice_key).unwrap();
    println!("Alice created! ID: {}", alice_account.id().to_bech32(miden_client::address::NetworkId::Testnet));

    // --- 3. DEPLOY FAUCET ---
    println!("\n[Step 2] Deploying MID Token Faucet...");
    let mut faucet_seed = [0u8; 32];
    client.rng().fill_bytes(&mut faucet_seed);
    let faucet_key = AuthSecretKey::new_falcon512_rpo();

    let faucet_account = AccountBuilder::new(faucet_seed)
        .account_type(AccountType::FungibleFaucet)
        .storage_mode(AccountStorageMode::Public)
        .with_auth_component(AuthFalcon512Rpo::new(faucet_key.public_key().to_commitment()))
        .with_component(BasicFungibleFaucet::new(
            TokenSymbol::new("MID").unwrap(), 
            8, 
            Felt::new(1_000_000)
        ).unwrap())
        .build()
        .unwrap();

    client.add_account(&faucet_account, false).await?;
    keystore.add_key(&faucet_key).unwrap();
    println!("Faucet deployed! ID: {}", faucet_account.id().to_bech32(miden_client::address::NetworkId::Testnet));

    // --- 4. MINT TOKENS ---
    println!("\n[Step 3] Minting 100 MID tokens for Alice...");
    let asset = FungibleAsset::new(faucet_account.id(), 100).unwrap();
    let mint_request = TransactionRequestBuilder::new()
        .build_mint_fungible_asset(
            asset, 
            alice_account.id(), 
            miden_client::note::NoteType::Public, 
            client.rng()
        ).unwrap();

    client.submit_new_transaction(faucet_account.id(), mint_request).await?;
    println!("Mint transaction submitted!");

    // --- 5. CONSUME TOKENS ---
    println!("Waiting for Testnet to process Alice's note...");
    loop {
        client.sync_state().await?;
        let notes = client.get_consumable_notes(Some(alice_account.id())).await?;
        
        if !notes.is_empty() {
            println!("Found {} notes! Consuming now...", notes.len());
            let consume_notes = notes.iter().map(|(n, _)| n.clone().try_into().unwrap()).collect();
            let consume_req = TransactionRequestBuilder::new().build_consume_notes(consume_notes).unwrap();
            client.submit_new_transaction(alice_account.id(), consume_req).await?;
            println!("Tokens successfully added to Alice's balance!");
            break;
        }
        tokio::time::sleep(Duration::from_secs(10)).await;
        println!("Still waiting for notes...");

    // --- STEP 6: DEPLOY COUNTER CONTRACT ---
    println!("\n[Step 4] Compiling and Deploying Counter Contract...");
    
    // 1. Load the MASM file we just created
    let counter_code = std::fs::read_to_string("masm/accounts/counter.masm")
        .expect("Make sure masm/accounts/counter.masm exists!");

    // 2. Define the storage slot (drawer) for the counter
    let counter_slot_name = miden_client::account::StorageSlotName::new("miden::tutorials::counter").unwrap();

    // 3. Compile the Assembly into a Miden Component
    let component_code = miden_client::assembly::CodeBuilder::new()
        .compile_component_code("external_contract::counter_contract", &counter_code)
        .unwrap();

    // 4. Create the component with one storage slot initialized to 0
    let counter_component = miden_client::account::AccountComponent::new(
        component_code,
        vec![miden_client::account::StorageSlot::with_value(counter_slot_name.clone(), [Felt::new(0); 4].into())],
    ).unwrap().with_supports_all_types();

    // 5. Generate a seed and build the Account
    let mut counter_seed = [0_u8; 32];
    client.rng().fill_bytes(&mut counter_seed);

    let counter_contract = AccountBuilder::new(counter_seed)
        .account_type(AccountType::RegularAccountImmutableCode)
        .storage_mode(AccountStorageMode::Public)
        .with_component(counter_component)
        .with_auth_component(miden_client::auth::NoAuth) // NoAuth means anyone can call this
        .build()
        .unwrap();

    // 6. Register the contract with the client
    client.add_account(&counter_contract, false).await.unwrap();

    println!("Success! Counter Contract Deployed.");
    println!("Contract ID: {}", counter_contract.id().to_bech32(miden_client::address::NetworkId::Testnet));    
    }

    Ok(())
}
</pre>

remember to always save your files

<img width="954" height="500" alt="image" src="https://github.com/user-attachments/assets/75ca0101-fe89-4712-a449-34aab657b4ce" />

after saving, its time to test our scripts
- close the terminal/powershell
- open a new terminal and run "wsl" for windows users
- navigate to your miden client directory

run, 
<pre>
  cd my-miden-client
</pre>

then, start script
<pre>
  cargo run
</pre>

<img width="1555" height="865" alt="image" src="https://github.com/user-attachments/assets/821fcce4-3906-40f8-a43b-d9469926208e" />

this should,
1. create alice account
2. deploy MID faucet tokens
3. mint MID faucet tokens
4. create and deploy the counter contract

this should be all for now,

in next guide, i will show how to interact with the miden counter contract by increasing states.

# gMIDEN




