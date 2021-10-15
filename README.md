﻿# CardanoSharp.Wallet 
[![Build status](https://ci.appveyor.com/api/projects/status/knh87k86mf7gbxyo?svg=true)](https://ci.appveyor.com/project/nothingalike/cardanosharp-wallet/branch/main) [![Test status](https://img.shields.io/appveyor/tests/nothingalike/cardanosharp-wallet)](https://ci.appveyor.com/project/nothingalike/cardanosharp-wallet/branch/main) [![NuGet Version](https://img.shields.io/nuget/v/CardanoSharp.Wallet.svg?style=flat)](https://www.nuget.org/packages/CardanoSharp.Wallet/) ![NuGet Downloads](https://img.shields.io/nuget/dt/CardanoSharp.Wallet.svg)

CardanoSharp Wallet is a .NET library for Creating/Managing Wallets and Building/Signing Transactions.


## Getting Started

CardanoSharp.Wallet is installed from NuGet.

```sh
Install-Package CardanoSharp.Wallet
```

## Generate a Mnemonic

The `KeyService` has operations which help for Generating and Restoring Mnemonics.

```cs
var keyService = new KeyService();
```

> KeyService is built for Dependency Injection and has a cooresponding interface: IKeyService.

Generate a Mnemonic: 

```cs
// This will generate a 24 English Mnemonic
Mnemonic mnemonic = keyService.Generate(24, WordLists.English);
```

Restore a Mnemonic: 
```cs
string words = "art forum devote street sure rather head chuckle guard poverty release quote oak craft enemy";
Mnemonic mnemonic = keyService.Restore(words);
```

## Create Private and Public Keys

Use powerful extensions to create and derive keys.

```csharp
// The rootKey is a PrivateKey made of up of the 
//  - byte[] Key
//  - byte[] Chaincode
PrivateKey rootKey = mnemonic.GetRootKey();

// This path will give us our Payment Key on index 0
string paymentPath = $"m/1852'/1815'/0'/0/0";
// The paymentPrv is Private Key of the specified path.
PrivateKey paymentPrv = rootKey.Derive(paymentPath);
// Get the Public Key from the Private Key
PublicKey paymentPub = paymentPrv.GetPublicKey(false);

// This path will give us our Stake Key on index 0
string stakePath = $"m/1852'/1815'/0'/2/0";
// The stakePrv is Private Key of the specified path
var stakePrv = rootKey.Derive(stakePath);
// Get the Public Key from the Stake Private Key
var stakePub = stakePrv.GetPublicKey(false);
```

## Create Addresses

The `AddressService` has operations which help Generating Addresses from Keys.

```cs
var addressService = new AddressService();
```

Using the same keys from the above when deriving our child keys, we can now get the public address.

```csharp
// Creating Addresses require the Public Payment and Stake Keys
Address baseAddr = addressService.GetAddress(
    paymentPub, 
    stakePub, 
    NetworkType.Testnet, 
    AddressType.Base);
```

If you already have an address.
```cs
Address baseAddr = new Address("addr_test1qz2fxv2umyhttkxyxp8x0dlpdt3k6cwng5pxj3jhsydzer3jcu5d8ps7zex2k2xt3uqxgjqnnj83ws8lhrn648jjxtwq2ytjqp");
```

## Fluent Derivation

We have a fluent api to help naivate the derivation paths.

```cs
// Restore a Mnemonic
var mnemonic = new KeyService().Restore(words);
var rootKey = mnemonic.GetRootKey();

// Fluent derivation API
var derivation = rootKey.Derive()   // IMasterNodeDerivation
    .Derive(PurposeType.Shelley)    // IPurposeNodeDerivation
    .Derive(CoinType.Ada)           // ICoinNodeDerivation
    .Derive(0)                      // IAccountNodeDerivation
    .Derive(RoleType.ExternalChain) // IRoleNodeDerivation
    //or .Derive(RoleType.Staking) 
    .Derive(0);                     // IIndexNodeDerivation

PrivateKey privateKey = derivation.PrivateKey;
PublicKey publicKey = derivation.PublicKey;
```

## Build and Sign Transactions

CardanoSharp.Wallet requires input from the chain in order to build transactions. Lets assume we have made a call to gather this information.

```cs 
uint currentSlot = 40000000;
ulong minFeeA = 44;
ulong minFeeB = 155381;
string inputTx = "0000000000000000000000000000000000000000000000000000000000000000";
```

Lets derive a few keys to use while building transactions.

```cs
// Derive down to our Account Node
var accountNode = rootKey.Derive()
    .Derive(PurposeType.Shelley)
    .Derive(CoinType.Ada)
    .Derive(0);

// Derive our Change Node on Index 0
var changeNode = accountNode
    .Derive(RoleType.InternalChain) 
    .Derive(0);

// Derive our Staking Node on Index 0
var stakingNode = accountNode
    .Derive(RoleType.Staking) 
    .Derive(0);

// Deriving our Payment Node
//  note: We did not derive down to the index.
var paymentNode = accountNode
    .Derive(RoleType.ExternalChain);
```

## Simple Transaction

Lets assume the following...
 - You have 100 ADA on path:        `m/1852'/1815'/0'/0/0`
 - You want to send 25 ADA to path: `m/1852'/1815'/0'/0/1`

### Build Transaction Body
```cs
// Generate the Reciever Address
Address paymentAddr = addressService.GetAddress(
    paymentNode.Derive(1).PublicKey, 
    stakingNode.PublicKey, 
    NetworkType.Testnet, 
    AddressType.Base);

// Generate an Address for changes
Address changeAddr = addressService.GetAddress(
    changeNode.PublicKey, 
    stakingNode.PublicKey, 
    NetworkType.Testnet, 
    AddressType.Base);

var transactionBody = TransactionBodyBuilder.Create
    .AddInput(inputTx, 0)
    .AddOutput(paymentAddr.GetBytes(), 25)
    .AddOutput(changeAddr.GetBytes(), 75)
    .SetTtl(currentSlot + 1000)
    .SetFee(0)
    .Build();
```

### Build Transaction Witnesses

For this simple transaction we really only need to add our keys. This is how we sign our transactions.
```cs
// Derive Sender Keys
var senderKeys = paymentNode.Derive(0);

var witnesses = TransactionWitnessSetBuilder.Create
    .AddVKeyWitness(senderKeys.PublicKey, senderKeys.PrivateKey);
```

### Calculate Fee 

```cs
// Construct Transaction Builder
var transactionBuilder = TransactionBuilder.Create
    .SetBody(transactionBody)
    .SetWitnesses(witnesses);

// Calculate Fee
var fee = transaction.CalculateFee(minFeeA, minFeeB);

// Update Fee and Rebuild
transactionBody.SetFee(fee);
Transaction transaction = transactionBuilder.Build();
transaction.TransactionBody.TransactionOutputs.Last().Value.Coin -= fee;
```

### More Examples

Please see the [Transaction Tests](https://github.com/CardanoSharp/cardanosharp-wallet/blob/main/CardanoSharp.Wallet.Test/TransactionTests.cs)
