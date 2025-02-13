---
title: Deploy a smart contract with CameLIGO
authors: 'John Joubert, Sasha Aldrick, Claude Barde, Tim McMackin'
last_update:
  date: 20 October 2023
---

This tutorial covers using the Octez command-line client to deploy a smart contract to Tezos.
The tutorial uses the LIGO programming language, which is one of the languages that you can write Tezos smart contracts in.
Specifically, this tutorial uses the CameLIGO version of LIGO, which has syntax similar to OCaml, but you don't need any experience with OCaml or LIGO to do this tutorial.

- If you are more familiar with JavaScript, try [Deploy a smart contract with jsLIGO](./jsligo).
- If you are more familiar with Python, try [Deploy a smart contract with SmartPy](./smartpy).
- To learn the Archetype language, try [Deploy a smart contract with Archetype](./archetype).

In this tutorial, you will learn how to:

- Connect the Octez client to a testnet
- Create a wallet
- Get tokens from a faucet
- Code a contract in LIGO, including:
  - Defining the storage for the contract
  - Defining entrypoints in the contract
  - Writing code to run when the entrypoints are called
- Deploy (or originate) the contract to Tezos and set its starting storage value
- Look up the current state of the contract
- Call the contract from the command line

## Tutorial contract

The contract that you deploy in this tutorial stores a single integer.
It provides entrypoints that clients can call to change the value of that integer:

- The `increment` endpoint accepts an integer as a parameter and adds that integer to the value in storage
- The `decrement` endpoint accepts an integer as a parameter and subtracts that integer to the value in storage
- The `reset` endpoint takes no parameters and resets the value in storage to 0

After you deploy the contract, you or any other user can call it through Octez or a distributed application (dApp).

## Prerequisites

To run this tutorial, you need the Octez client and LIGO.

- To install the LIGO programming language, version 1.0.0 or later, see https://ligolang.org/docs/intro/installation.
You can verify that LIGO is installed by running this command:

   ```bash
   ligo version
   ```

   If you see a message with a version 1.0.0 or later, LIGO is installed correctly.

- To install the Octez client, which allows you to send transactions to the Tezos blockchain, use your operating system's package manager:

   - For MacOS, run these commands:

   ```bash
   brew tap serokell/tezos-packaging-stable https://github.com/serokell/tezos-packaging-stable.git
   brew install tezos-client
   ```

   - For Ubuntu, Windows WSL, and Linux distributions that use `apt`, run these commands:

   ```bash
   REPO="ppa:serokell/tezos"
   sudo add-apt-repository -y $REPO && sudo apt-get update
   sudo apt-get install -y tezos-client
   ```

   - For Fedora and Linux distributions that use Copr, run these commands:

   ```bash
   REPO="@Serokell/Tezos"
   dnf copr enable -y $REPO && dnf update -y
   dnf install -y tezos-client
   ```

   You can verify that the Octez client is installed by running this command:

   ```bash
   octez-client --version
   ```

   If you see a message with the version of Octez that you have installed, the Octez client is installed correctly.
   For help on Octez, run `octez-client --help` or see http://tezos.gitlab.io/index.html.

   For more detailed installation instructions, see [How to get Tezos](http://tezos.gitlab.io/introduction/howtoget.html).

LIGO is a high-level programming language created by Marigold to write smart contracts for the Tezos blockchain.

It abstracts away the complexity of using Michelson (the smart contract language directly available on-chain) and provides different syntaxes that make it easier to write smart contracts on Tezos.

LIGO provides two syntaxes: *jsLigo*, a syntax similar to TypeScript, and *CameLigo*, a syntax similar to OCaml.
This tutorial uses CameLigo, but you do not need any experience with OCaml to run it.

## Create a project folder

Follow these steps to create a LIGO project:

1. On the command-line terminal, create a folder for the project and open it.
You can name your project anything you want, such as `example-smart-contract-cameligo`.

   ```bash
   mkdir example-smart-contract-cameligo
   cd example-smart-contract-cameligo
   ```

1. Create a file named `increment.mligo` in the project folder.
This is where the contract code goes.

   ```bash
   touch increment.mligo
   ```

## Switch to a testnet

Before you deploy your contract to the main Tezos network (referred to as *Mainnet*), you can deploy it to a testnet.
Testnets are useful for testing Tezos operations because testnets provide tokens for free so you can work with them without spending real tokens.

Tezos testnets are listed on this site: https://teztnets.xyz/.

The [Ghostnet](https://teztnets.xyz/ghostnet-about) testnet is a good choice for testing because it is intended to be long-lived, as opposed to shorter-term testnets that allow people to test new Tezos features.

Follow these steps to set your Octez client to use a testnet instead of the main network:

1. On https://teztnets.xyz/, click the testnet to use, such as Ghostnet.

1. Copy the one of the testnet's public RPC endpoints, such as `https://rpc.ghostnet.teztnets.xyz`.

1. Set your Octez client to use this testnet by running this command on the command line, replacing the testnet RPC URL with the URL that you copied:

   ```bash
   octez-client --endpoint https://rpc.ghostnet.teztnets.xyz config update
   ```

   Octez shows a warning that you are using a testnet instead of Mainnet.

1. Verify that you are using a testnet by running this command:

   ```bash
   octez-client config show
   ```

   The response from Octez includes the URL of the testnet.

## Create a local wallet

Deploying and using a smart contract costs fees, so you need a local wallet and tez tokens.
The Octez client can manage a local wallet for you, and you can get tez tokens on testnets from faucets.

1. Run the following command to generate a local wallet, replacing `local_wallet` with a name for your wallet:

   ```bash
   octez-client gen keys local_wallet
   ```

1. Get the address for the wallet by running this command, again replacing `local_wallet` with the name of your local wallet.

   ```bash
   octez-client show address local_wallet
   ```

   The Octez client prints a warning that you are using a testnet and the address of the new wallet in the `hash` field.
   The wallet address begins with `tz1`, `tz2`, or `tz3`, as in this example:

   ```bash
   Warning:

                    This is NOT the Tezos Mainnet.

              Do NOT use your fundraiser keys on this network.

   Hash: tz1dW9Mk...........H67L
   Public Key: edp.............................bjbeDj
   ```

   You need the wallet address to send funds to the wallet, to deploy the contract, and to send transactions to the contract.

1. On the testnets page at https://teztnets.xyz/, click the faucet link for the testnet you are using.
For example, the Ghostnet faucet is at https://faucet.ghostnet.teztnets.xyz.

1. On the faucet page, paste your wallet address into the input field labeled "Or fund any address" and click the button for the amount of tez to add to your wallet.
20 tez is enough to work with the tutorial application, and you can return to the faucet later if you need more tez.
It may take a few minutes for the faucet to send the tokens and for those tokens to appear in your wallet.

   You can use the faucet as much as you need to get tokens on the testnet, but those tokens are worthless and cannot be used on Mainnet.

   ![Fund your wallet using the Ghostnet Faucet](/img/tutorials/wallet-funding.png)

1. Run this command to check the balance of your wallet:

   ```bash
   octez-client get balance for local_wallet
   ```

If your wallet is set up correctly and the faucet has sent tokens to it, the Octez client prints the balance of your wallet, as in this example:

```
20 ꜩ
```

## Create the contract

The contract that you will create has these basic parts:

- A type that describes the contract's storage, in this case an integer.
The storage can be a primitive type such as an integer, string, or timestamp, or a complex data type that contains multiple values.
For more information on contract data types, see [Data types](../../smart-contracts/data-types).

- Internal functions called entrypoints that run code when clients call the contract.

- A type that describes the return value of the entrypoints.

Follow these steps to create the code for the contract:

1. Open the `increment.mligo` file in any text editor.

1. Add this line of code to set the storage type to an integer:

   ```ocaml
   type storage = int
   ```

1. Add this code to define the return type for the endpoints.
Tezos entrypoints return two values: a list of other operations to call and the new value of the contract's storage.

   ```ocaml
   type returnValue = operation list * storage
   ```

1. Add the code for the increment and decrement entrypoints:

   ```ocaml
   // Increment entrypoint
   [@entry] let increment (delta : int) (store : storage) : returnValue =
     [], store + delta

   // Decrement entrypoint
   [@entry] let decrement (delta : int) (store : storage) : returnValue =
     [], store - delta
   ```

   These functions begin with the `@entry` annotation to indicate that they are entrypoints.
   They accept two parameters: the change in the storage value (an integer) and the current value of the storage (in the `storage` type that you created earlier in the code).
   They return a value of the type `returnValue` that you created in the previous step.

   Each function returns an empty list of other operations to call and the new value of the storage.

1. Add this code for the reset entrypoint:

   ```ocaml
   // Reset entrypoint
   [@entry] let reset (() : unit) (_ : storage) : returnValue =
     [], 0
   ```

   This function is similar to the others, but it does not take the current value of the storage into account.
   It always returns an empty list of operations and 0.

The complete contract code looks like this:

```ocaml
type storage = int

type returnValue = operation list * storage

// Increment entrypoint
[@entry] let increment (delta : int) (store : storage) : returnValue =
  [], store + delta

// Decrement entrypoint
[@entry] let decrement (delta : int) (store : storage) : returnValue =
  [], store - delta

// Reset entrypoint
[@entry] let reset (() : unit) (_ : storage) : returnValue =
  [], 0
```

## Testing and compiling the contract

Before you can deploy the contract to Tezos, you must compile it to Michelson, the base language of Tezos contracts.

1. Test the contract by passing parameters and the storage value to the LIGO `dry-run` command.
For example, this command sets the storage at 10 and increments it by 32:

   ```bash
   ligo run dry-run increment.mligo "Increment(32)" "10"
   ```

   The terminal should show the response `(LIST_EMPTY(), 42)`.
   This response means that the contract did not call any other contracts, so the list of operations is empty.
   Then it shows the new value of the storage.
   You can test the decrement and reset functions in the same way.

1. Run this command to compile the contract:

   ```bash
   ligo compile contract increment.mligo -o increment.tz
   ```

   If the compilation succeeds, no messages are shown in the terminal.
   If you see error messages, verify that your contract code matches the code in the previous section.

Now you can deploy the contract.

## Deploying (originating) to the testnet

Deploying a contract to the network is called "originating."
Originating the contract requires a small amount of Tezos tokens as a fee.

1. Run the following command to originate the smart contract, changing `$MY_TZ_ADDRESS` to the address or local name of the wallet that you created earlier in the tutorial:

   ```bash
   octez-client originate contract my-counter \
       transferring 0 from $MY_TZ_ADDRESS \
       running increment.tz \
       --init 10 --burn-cap 0.1 --force
   ```

   This command includes these parts:

     - It uses the Octez client `originate contract` command to originate the contract and assigns the local name `my-counter` to the contract
     - It includes 0 tokens from your wallet with the transaction, but the `--burn-cap` argument allows the transaction to take up to 0.1 tez from your wallet for fees.
     - It sets the initial value of the contract storage to 10 with the `--init` argument.

   If the contract deploys successfully, Octez shows the address of the new contract, as in this example:

   ```bash
   New contract KT1Nnk.................UFsJrq originated.
   The operation has only been included 0 blocks ago.
   We recommend to wait more.
   ```

1. Copy the contract address, which starts with `KT1`.

1. Optional: Run the command `octez-client get balance for local_wallet` to get the updated balance of your wallet.

1. Verify that the contract deployed successfully by finding it on a block explorer:

  1. Open a Tezos block explorer such as [TzKT](https://tzkt.io) or [Better Call Dev](https://better-call.dev/).

  1. Set the explorer to Ghostnet instead of Mainnet.

  1. Paste the contract address, which starts with `KT1`, into the search field and press Enter.

  1. Go to the Storage tab to see that the initial value of the storage is 10.

## Calling the contract

Now you can call the contract from any Tezos client, including Octez.

To increment the current storage by a certain value, call the `increment` entrypoint, as in this example, again changing `$MY_TZ_ADDRESS` to the address or local name of the wallet that you created earlier in the tutorial:

```bash
octez-client --wait none transfer 0 from $MY_TZ_ADDRESS to my-counter --entrypoint 'increment' --arg '5' --burn-cap 0.1
```

The previous example uses the local name `my-counter`.
You can also specify the contract address.

To decrement the current storage by a certain value, call the `decrement` entrypoint, as in this example:

```bash
octez-client --wait none transfer 0 from $MY_TZ_ADDRESS to my-counter --entrypoint 'decrement' --arg '6' --burn-cap 0.1
```

Finally, to reset the current storage to zero, call the `reset` entrypoint, as in this example:

```bash
octez-client --wait none transfer 0 from $MY_TZ_ADDRESS to my-counter --entrypoint 'reset' --arg 'Unit' --burn-cap 0.1
```

You can go back to the block explorer to verify that the storage of the contract changed.

## Summary

Now the contract is running on the Tezos blockchain.
You or any other user can call it from any source that can send transactions to Tezos, including Octez, dApps, and other contracts.

If you want to continue working with this contract, try creating a dApp to call it from a web application, similar to the dApp that you create in the tutorial [Build your first app on Tezos](../build-your-first-app/).
You can also try adding your own endpoints and originating a new contract, but you cannot update the existing contract after it is deployed.
