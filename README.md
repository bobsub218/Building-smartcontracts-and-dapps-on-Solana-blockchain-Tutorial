# Building-smartcontracts-and-dapps-on-Solana-blockchain

<img width="1034" alt="solana-logo" src="https://user-images.githubusercontent.com/73669069/134777355-a2f66222-e5a2-45b3-ba67-1ebe7c50b6b6.png">

# Introduction
Solana is a web-scale, open-source blockchain protocol that supports developers and institutions around the world to build decentralized applications (DApps) and marketplaces. Solana’s protocol is fast, secure, and censorship-resistant, which provides the flexibility of an open infrastructure required to build applications for mass adoption.

This high-performance blockchain provides the fully decentralized, secure, and highly scalable infrastructure necessary for tomorrow’s DApps and decentralized marketplaces. It leverages a set of breakthrough computational technologies that can support thousands of nodes, allowing for transaction throughput to proportionally scale with network bandwidth.
# Prerequisites
In this tutorial we will develop a smart contract in Rust then compile it with Cargo and deploy it using NodeJS.
# Requirements
* Knowledge of [Rust](https://www.rust-lang.org/learn)
* Knowledge NodeJS
* [Solana / web3.js](https://www.npmjs.com/package/@solana/web3.js) module
# Body of Tutorial
This Solana tutorial goes through a step by step process of setting up a development environment for Solana, writing and deploying smart contracts.

I settled on setting up a WSL (Windows Subsystem For Linux) Ubuntu version so that I could write the code in Windows and then just use a linux command line to compile the Rust smart contract to a .so file.

If you don’t have WSL set up already it’s a great tool and there are detailed instructions and a download link here: https://docs.microsoft.com/en-us/windows/wsl/install-win10

Once installed I setup nodejs, npm, python, rust and the solana sdk.
Here are the commands:
```
apt upgrade
apt update
apt install nodejs
apt install npm
apt install python3-pip
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
sh -c "$(curl -sSfL https://release.solana.com/v1.5.7/install)"
source $HOME/.cargo/env
export PATH="/root/.local/share/solana/install/active_release/bin:$PATH"
export RUST_LOG=solana_runtime::system_instruction_processor=trace,solana_runtime::message_processor=debug,solana_bpf_loader=debug,solana_rbpf=debug
solana-test-validator --log 
```

Once setup we can test it by running a simple hello world application
```
git clone https://github.com/solana-labs/example-helloworld.git
cd example-helloworld/
npm install
npm run build:program-rust
```
Recommend is installing the Rust extension for VSCode or whatever editor you are using.

### First Solana Smart Contract
I’d recommend taking a look at some of the example code provided by Solana labs here: https://github.com/solana-labs

There’s a good introduction to coding with Rust here: https://www.rust-lang.org/learn

Read everything and refer back to the Solana developer documentation here: https://docs.solana.com/developing/programming-model/overview

Let’s start with the hello world example and disect what each part does. The full code can be found at: https://github.com/solana-labs/example-helloworld/blob/master/src/program-rust/src/lib.rs
```
use byteorder::{ByteOrder, LittleEndian};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};
use std::mem;
entrypoint!(process_instruction);
```
We start by doing some standard includes and declaring an entry point which will be the process_instruction function.

```
fn process_instruction(
program_id: &Pubkey,
accounts: &[AccountInfo],
_instruction_data: &[u8],
```
The program_id is the public key where the contract is stored and the accountInfo is the appAccount to say hello to.

```
) -> ProgramResult {
msg!("Helloworld Rust program entrypoint");
let accounts_iter = &mut accounts.iter();
let account = next_account_info(accounts_iter)?;
```
The ProgramResult is where the magic happens and we start by printing a fairly pointless message and then select the accountInfo by looping through although in practice there is likely only one.

```if account.owner != program_id {
  msg!("Greeted account does not have the correct program id");
  return Err(ProgramError::IncorrectProgramId);
}
```

Security check to see if the account owner has permission.

```
if account.try_data_len()? < mem::size_of::<u32>() {
  msg!("Greeted account data length too small for u32");
  return Err(ProgramError::InvalidAccountData);
}
```

Check to see if data is available to store a u32 integar.

```
let mut data = account.try_borrow_mut_data()?;
let mut num_greets = LittleEndian::read_u32(&data);
num_greets += 1;
LittleEndian::write_u32(&mut data[0..], num_greets);
msg!("Hello!");
Ok(())
```
Finally we get to the good stuff where we “borrow” the existing appAccount data, increase that value by one and rewrite it back. Then last of course it outputs another fairly pointless “Hello” message.

So the program increments the number stored in accountInfo.data.num_greets by one each time it is run.

### Deploying A Contract With NodeJS

You can deploy contracts via the command line but I found it much easier to have a script which takes a compiled .so file and deploys it.

I used NodeJS and the @solana/web3.js module for this.

```
const connection = new 
solanaWeb3.Connection('https://testnet.solana.com');
 
const payerAccount = new solanaWeb3.Account();

const res = await connection.requestAirdrop(payerAccount.publicKey, 1000000000);
```

So the first step is to setup a connection to the testnet. Create a user which will be used to pay for all the transactions. Then we will use the requestAirdrop function to request 1 SOL (1000000000 Lamports). Because it’s the testnet we will get some SOL tokens deposited to our account.

To deploy a contract on mainnet we would purchase some SOL tokens from FTX and then send them to a Sollet.io account. The private key could be exported from there and imported as a buffer array like so.
```
const payerAccount = new solanaWeb3.Account([1,185,72,49,215,81,171,50,85,54,122,53,24,248,3,221,42,85,82,43,128,80,215,127,68,99,172,141,116,237,232,85,185,31,141,73,173,222,173,174,4,212,0,104,157,80,63,147,21,81,140,201,113,76,156,161,154,92,70,67,163,52,219,72]);
```
So we have a private key / public key pair for an account with some funds. The next step is to upload the smart contract code to a separate account.
```
const data = await fs.readFile('./directory/file.so');
const programAccount = new solanaWeb3.Account();
await solanaWeb3.BpfLoader.load(
  connection,
  payerAccount,
  programAccount,
  data,
  solanaWeb3.BPF_LOADER_PROGRAM_ID,
);
const programId = programAccount.publicKey;
console.log('Program loaded to account', programId.toBase58());
```
We read the file.so that we created using linux to compile our rust smart contract. 
This is usually in ./target/deploy/whatever.so

We create another account for the programAccount and load the code, paying for the transaction and storage costs using the payerAccount we previously set up.

Finally we print out the base58 version of the programId which is just the public key of the programAccount. The next step is to create a 3rd account for the app itself which will store any data required for the dApp.

```
const appAccount = new solanaWeb3.Account();
const appPubkey = appAccount.publicKey;
console.log('Creating app account', appPubkey.toBase58());
const space = smartContract.dataLayout.span;
const lamports = 5000000000;
const transaction = new solanaWeb3.Transaction().add(
  solanaWeb3.SystemProgram.createAccount({
    fromPubkey: payerAccount.publicKey,
    newAccountPubkey: appPubkey,
    lamports,
    space,
    programId,
  }),
);
await solanaWeb3.sendAndConfirmTransaction(
  connection,
  transaction,
  [payerAccount, appAccount],
  {
    commitment: 'singleGossip',
    preflightCommitment: 'singleGossip',
  },
);
```

One quirk here is that we have to specify how much space is required for data when setting up the account. I used the [buffer-layout](https://www.npmjs.com/package/buffer-layout) module to map and calculate this for the space variable. For simple contracts this probably isn’t necessary.

Note that this space is rented not purchased and the appAccount will need some SOL to pay the rental fees.
```
const buffer = Buffer.from('hello solana', 'utf8');
const instruction = new solanaWeb3.TransactionInstruction({
  keys: [{pubkey: app.appAccount.publicKey, isSigner: false, isWritable: true}],
  programId: app.programId,
  data: buffer,
});
const confirmation = await solanaWeb3.sendAndConfirmTransaction(
  connection,
  new solanaWeb3.Transaction().add(instruction),
  [payerAccount],
  {
    commitment: 'singleGossip',
    preflightCommitment: 'singleGossip',
  },
);
```
Finally we are going to interact with our smart contract by sending it some data. The code above creates a buffer from a string. Puts this into an instruction which is then sent using the sendAndConfirmTransaction function.
This is sent from the original payerAccount which hopefully still has some funds left.

# Conclusion
