# go-algorand-sdk

[![Build Status](https://travis-ci.com/algorand/go-algorand-sdk.svg?branch=master)](https://travis-ci.com/algorand/go-algorand-sdk)
[![Go Report Card](https://goreportcard.com/badge/github.com/algorand/go-algorand-sdk)](https://goreportcard.com/report/github.com/algorand/go-algorand-sdk)
[![GoDoc](https://godoc.org/github.com/algorand/go-algorand-sdk?status.svg)](https://godoc.org/github.com/algorand/go-algorand-sdk)

The Algorand golang SDK provides:

- HTTP clients for the algod (agreement) and kmd (key management) APIs
- Standalone functionality for interacting with the Algorand protocol, including transaction signing, message encoding, etc.

# Documentation

Full documentation is available [on godoc](https://godoc.org/github.com/algorand/go-algorand-sdk). You can also self-host the documentation by running `godoc -http=:8099` and visiting `http://localhost:8099/pkg/github.com/algorand/go-algorand-sdk` in your web browser.

Additional developer documentation can be found on [developer.algorand.org](https://developer.algorand.org/)

# Package overview

In `client/`, the `algod` and `kmd` packages provide HTTP clients for their corresponding APIs. `algod` is the Algorand protocol daemon, responsible for reaching consensus with the network and participating in the Algorand protocol. You can use it to check the status of the blockchain, read a block, look at transactions, or submit a signed transaction. `kmd` is the key management daemon. It is responsible for managing spending key material, signing transactions, and managing wallets.

`types` contains the data structures you'll use when interacting with the network, including addresses, transactions, multisig signatures, etc. Some types (like `Transaction`) have their own packages containing constructors (like `MakePaymentTxn`).

`encoding` contains the `json` and `msgpack` packages, which can be used to serialize messages for the algod/kmd APIs and the network.

`mnemonic` contains support for turning 32-byte keys into checksummed, human-readable mnemonics (and going from mnemonics back to keys).

# SDK Development

Run tests with `make docker-test`

# Quick Start

To download the SDK, open a terminal and use the `go get` command.

```command
go get -u github.com/algorand/go-algorand-sdk/...
```

If you are connected to the Algorand network, your algod process should already be running. The kmd process must be started manually, however. Start and stop kmd using `goal kmd start` and `goal kmd stop`:

```command
goal kmd start -d <your-data-directory>
```

Here's a simple example which creates clients for algod and kmd:

```golang
package main

import (
	"fmt"

	"github.com/algorand/go-algorand-sdk/client/v2/algod"
	"github.com/algorand/go-algorand-sdk/client/kmd"
)

var (
    algodAddress = "http://localhost:4001"
    algodToken = strings.Repeat("a", 64)  // contents of algod.token

    kmdAddress = "http://localhost:4002"
    kmdToken = strings.Repeat("a", 64)// contents of kmd.token
)

func main() {
	// Create an algod client
	algodClient, err := algod.MakeClient(algodAddress, algodToken)
	if err != nil {
		return
	}

	// Create a kmd client
	kmdClient, err := kmd.MakeClient(kmdAddress, kmdToken)
	if err != nil {
		return
	}

	fmt.Printf("algod: %T, kmd: %T\n", algodClient, kmdClient)
}
```

# Building sources

```
make build
```

The build process includes generating some files. Refer to `Makefile` for details.

# Examples

## algod client

Here is an example that creates an algod client and uses it to fetch node status information, and then a specific block.

```golang
package main

import (
	"encoding/json"
	"fmt"

	"github.com/algorand/go-algorand-sdk/client/v2/algod"
)

// These constants represent the algod REST endpoint and the corresponding
// API token. You can retrieve these from the `algod.net` and `algod.token`
// files in the algod data directory.
var (
    algodAddress = "http://localhost:4001"
    algodToken = strings.Repeat("a", 64)
)

func main() {
	// Create an algod client
	algodClient, err := algod.MakeClient(algodAddress, algodToken)
	if err != nil {
		fmt.Printf("failed to make algod client: %s\n", err)
		return
	}

	// Print algod status
	nodeStatus, err := algodClient.Status().Do(context.Background())
	if err != nil {
		fmt.Printf("error getting algod status: %s\n", err)
		return
	}

	fmt.Printf("algod last round: %d\n", nodeStatus.LastRound)
	fmt.Printf("algod time since last round: %d\n", nodeStatus.TimeSinceLastRound)
	fmt.Printf("algod catchup: %d\n", nodeStatus.CatchupTime)
	fmt.Printf("algod latest version: %s\n", nodeStatus.LastVersion)

	// Fetch block information
	lastBlock, err := algodClient.Block(nodeStatus.LastRound).Do(context.Background())
	if err != nil {
		fmt.Printf("error getting last block: %s\n", err)
		return
	}

	// Print the block information
	fmt.Printf("\n-----------------Block Information-------------------\n")
	blockJSON, err := json.MarshalIndent(lastBlock, "", "\t")
	if err != nil {
		fmt.Printf("Can not marshall block data: %s\n", err)
	}
	fmt.Printf("%s\n", blockJSON)
}
```

## kmd client

The following example creates a wallet, and generates an account within that wallet.

```golang
package main

import (
	"fmt"

	"github.com/algorand/go-algorand-sdk/client/kmd"
	"github.com/algorand/go-algorand-sdk/types"
)

// These constants represent the kmdd REST endpoint and the corresponding API
// token. You can retrieve these from the `kmd.net` and `kmd.token` files in
// the kmd data directory.
var (
    kmdAddress = "http://localhost:4002"
    kmdToken = strings.Repeat("a", 64)
)

func main() {
	// Create a kmd client
	kmdClient, err := kmd.MakeClient(kmdAddress, kmdToken)
	if err != nil {
		fmt.Printf("failed to make kmd client: %s\n", err)
		return
	}
	fmt.Println("Made a kmd client")

	// Create the example wallet, if it doesn't already exist
	cwResponse, err := kmdClient.CreateWallet("testwallet", "testpassword", kmd.DefaultWalletDriver, types.MasterDerivationKey{})
	if err != nil {
		fmt.Printf("error creating wallet: %s\n", err)
		return
	}

	// We need the wallet ID in order to get a wallet handle, so we can add accounts
	exampleWalletID := cwResponse.Wallet.ID
	fmt.Printf("Created wallet '%s' with ID: %s\n", cwResponse.Wallet.Name, exampleWalletID)

	// Get a wallet handle. The wallet handle is used for things like signing transactions
	// and creating accounts. Wallet handles do expire, but they can be renewed
	initResponse, err := kmdClient.InitWalletHandle(exampleWalletID, "testpassword")
	if err != nil {
		fmt.Printf("Error initializing wallet handle: %s\n", err)
		return
	}

	// Extract the wallet handle
	exampleWalletHandleToken := initResponse.WalletHandleToken

	// Generate a new address from the wallet handle
	genResponse, err := kmdClient.GenerateKey(exampleWalletHandleToken)
	if err != nil {
		fmt.Printf("Error generating key: %s\n", err)
		return
	}
	fmt.Printf("Generated address %s\n", genResponse.Address)
}
```

This account can now be used to sign transactions, but you will need some funds to get started. If you are on the test network, you can use the [dispenser](https://bank.testnet.algorand.network) to seed your account with some Algos.

## Backing up a Wallet

You can export a master derivation key from the wallet and convert it to a mnemonic phrase in order to back up any generated addresses. This backup phrase will only allow you to recover wallet-generated keys; if you import an external key into a kmd-managed wallet, you'll need to back up that key by itself in order to recover it.

```golang
package main

import (
	"fmt"

	"github.com/algorand/go-algorand-sdk/client/kmd"
	"github.com/algorand/go-algorand-sdk/mnemonic"
)

// These constants represent the kmd REST endpoint and the corresponding API
// token. You can retrieve these from the `kmd.net` and `kmd.token` files in
// the kmd data directory.
var (
	kmdAddress = "http://localhost:4002"
	kmdToken = strings.Repeat("a",64)
)

func main() {
	// Create a kmd client
	kmdClient, err := kmd.MakeClient(kmdAddress, kmdToken)
	if err != nil {
		fmt.Printf("failed to make kmd client: %s\n", err)
		return
	}
	fmt.Println("Made a kmd client")

	// Get the list of wallets
	listResponse, err := kmdClient.ListWallets()
	if err != nil {
		fmt.Printf("error listing wallets: %s\n", err)
		return
	}

	// Find our wallet name in the list
	var exampleWalletID string
	fmt.Printf("Got %d wallet(s):\n", len(listResponse.Wallets))
	for _, wallet := range listResponse.Wallets {
		fmt.Printf("ID: %s\tName: %s\n", wallet.ID, wallet.Name)
		if wallet.Name == "testwallet" {
			fmt.Printf("found wallet '%s' with ID: %s\n", wallet.Name, wallet.ID)
			exampleWalletID = wallet.ID
		}
	}

	// Get a wallet handle
	initResponse, err := kmdClient.InitWalletHandle(exampleWalletID, "testpassword")
	if err != nil {
		fmt.Printf("Error initializing wallet handle: %s\n", err)
		return
	}

	// Extract the wallet handle
	exampleWalletHandleToken := initResponse.WalletHandleToken

	// Get the backup phrase
	exportResponse, err := kmdClient.ExportMasterDerivationKey(exampleWalletHandleToken, "testpassword")
	if err != nil {
		fmt.Printf("Error exporting backup phrase: %s\n", err)
		return
	}
	mdk := exportResponse.MasterDerivationKey

	// This string should be kept in a safe place and not shared
	stringToSave, err := mnemonic.FromKey(mdk[:])
	if err != nil {
		fmt.Printf("Error getting backup phrase: %s\n", err)
		return
	}

	fmt.Printf("Backup Phrase: %s\n", stringToSave)
}
```

To restore a wallet, convert the phrase to a key and pass it to `CreateWallet`. This call will fail if the wallet already exists:

```golang
package main

import (
	"fmt"

	"github.com/algorand/go-algorand-sdk/client/kmd"
	"github.com/algorand/go-algorand-sdk/mnemonic"
	"github.com/algorand/go-algorand-sdk/types"
)

// These constants represent the kmd REST endpoint and the corresponding API
// token. You can retrieve these from the `kmd.net` and `kmd.token` files in
// the kmd data directory.
var (
	 kmdAddress = "http://localhost:4002"
	 kmdToken = strings.Repeat("a", 4001)
)

func main() {
	// Create a kmd client
	kmdClient, err := kmd.MakeClient(kmdAddress, kmdToken)
	if err != nil {
		fmt.Printf("failed to make kmd client: %s\n", err)
		return
	}
	backupPhrase := "fire enlist diesel stamp nuclear chunk student stumble call snow flock brush example slab guide choice option recall south kangaroo hundred matrix school above zero"
	keyBytes, err := mnemonic.ToKey(backupPhrase)
	if err != nil {
		fmt.Printf("failed to get key: %s\n", err)
		return
	}

	var mdk types.MasterDerivationKey
	copy(mdk[:], keyBytes)
	cwResponse, err := kmdClient.CreateWallet("testwallet", "testpassword", kmd.DefaultWalletDriver, mdk)
	if err != nil {
		fmt.Printf("error creating wallet: %s\n", err)
		return
	}
	fmt.Printf("Created wallet '%s' with ID: %s\n", cwResponse.Wallet.Name, cwResponse.Wallet.ID)
}
```

## Signing and submitting a transaction

The following example shows how to to use both KMD and Algod when signing and submitting a transaction. You can also sign a transaction offline, which is shown in the next section of this document.

```golang
package main

import (
	"context"
	"fmt"
	"strings"

	"github.com/algorand/go-algorand-sdk/client/kmd"
	"github.com/algorand/go-algorand-sdk/client/v2/algod"
	"github.com/algorand/go-algorand-sdk/future"
)

// CHANGE ME
var (
	algodAddress = "http://localhost:4001"
	algodToken   = strings.Repeat("a", 64)

	kmdAddress = "http://localhost:4002"
	kmdToken   = strings.Repeat("a", 64)
)

func main() {
	// Create a kmd client
	kmdClient, err := kmd.MakeClient(kmdAddress, kmdToken)
	if err != nil {
		fmt.Printf("failed to make kmd client: %s\n", err)
		return
	}
	fmt.Println("Made a kmd client")

	// Create an algod client
	algodClient, err := algod.MakeClient(algodAddress, algodToken)
	if err != nil {
		fmt.Printf("failed to make algod client: %s\n", err)
		return
	}
	fmt.Println("Made an algod client")

	// Get the list of wallets
	listResponse, err := kmdClient.ListWallets()
	if err != nil {
		fmt.Printf("error listing wallets: %s\n", err)
		return
	}

	// Find our wallet name in the list
	var exampleWalletID string
	fmt.Printf("Got %d wallet(s):\n", len(listResponse.Wallets))
	for _, wallet := range listResponse.Wallets {
		fmt.Printf("ID: %s\tName: %s\n", wallet.ID, wallet.Name)
		if wallet.Name == "testwallet" {
			fmt.Printf("found wallet '%s' with ID: %s\n", wallet.Name, wallet.ID)
			exampleWalletID = wallet.ID
			break
		}
	}
	// Get a wallet handle
	initResponse, err := kmdClient.InitWalletHandle(exampleWalletID, "testpassword")
	if err != nil {
		fmt.Printf("Error initializing wallet handle: %s\n", err)
		return
	}

	// Extract the wallet handle
	exampleWalletHandleToken := initResponse.WalletHandleToken

	// Generate a new address from the wallet handle
	gen1Response, err := kmdClient.GenerateKey(exampleWalletHandleToken)
	if err != nil {
		fmt.Printf("Error generating key: %s\n", err)
		return
	}
	fmt.Printf("Generated address 1 %s\n", gen1Response.Address)
	fromAddr := gen1Response.Address

	// Generate a new address from the wallet handle
	gen2Response, err := kmdClient.GenerateKey(exampleWalletHandleToken)
	if err != nil {
		fmt.Printf("Error generating key: %s\n", err)
		return
	}
	fmt.Printf("Generated address 2 %s\n", gen2Response.Address)
	toAddr := gen2Response.Address

	// Get the suggested transaction parameters
	params, err := algodClient.SuggestedParams().Do(context.Background())
	if err != nil {
		fmt.Printf("error getting suggested tx params: %s\n", err)
		return
	}

	// Make transaction
	tx, err := future.MakePaymentTxn(fromAddr, toAddr, 1000, nil, "", params)
	if err != nil {
		fmt.Printf("Error creating transaction: %s\n", err)
		return
	}

	// Sign the transaction
	signResponse, err := kmdClient.SignTransaction(exampleWalletHandleToken, "testpassword", tx)
	if err != nil {
		fmt.Printf("Failed to sign transaction with kmd: %s\n", err)
		return
	}

	fmt.Printf("kmd made signed transaction with bytes: %x\n", signResponse.SignedTransaction)

	// Broadcast the transaction to the network
	// Note that this transaction will get rejected because the accounts do not have any tokens
	txid, err := algodClient.SendRawTransaction(signResponse.SignedTransaction).Do(context.Background())
	if err != nil {
		fmt.Printf("failed to send transaction: %s\n", err)
		return
	}

	fmt.Printf("Transaction ID: %s\n", txid)
}
```

## Sign a transaction offline

The following example shows how to create a transaction and sign it offline. You can also create the transaction online and then sign it offline.

```golang
package main

import (
	"fmt"
	"io/ioutil"

	"github.com/algorand/go-algorand-sdk/crypto"
	"github.com/algorand/go-algorand-sdk/future"
	"github.com/algorand/go-algorand-sdk/mnemonic"
	"github.com/algorand/go-algorand-sdk/types"
)

func main() {

	account := crypto.GenerateAccount()
	fmt.Printf("account address: %s\n", account.Address)

	m, err := mnemonic.FromPrivateKey(account.PrivateKey)
	fmt.Printf("backup phrase = %s\n", m)

	params, err := algodClient.SuggestedParams().Do(context.Background())
	if err != nil {
		fmt.Printf("error getting suggested tx params: %s\n", err)
		return
	}
	tx, err := future.MakePaymentTxn(
		account.Address.String(), "4MYUHDWHWXAKA5KA7U5PEN646VYUANBFXVJNONBK3TIMHEMWMD4UBOJBI4",
		amount, nil, "", params,
	)
	if err != nil {
		fmt.Printf("Error creating transaction: %s\n", err)
		return
	}
	fmt.Printf("Made unsigned transaction: %+v\n", tx)
	fmt.Println("Signing transaction with go-algo-sdk library function (not kmd)")

	// Sign the Transaction
	txid, bytes, err := crypto.SignTransaction(account.PrivateKey, tx)
	if err != nil {
		fmt.Printf("Failed to sign transaction: %s\n", err)
		return
	}

	// Save the signed object to disk
	fmt.Printf("Made signed transaction with TxID %s\n", txid)
	filename := "./signed.tx"
	err = ioutil.WriteFile(filename, bytes, 0644)
	if err != nil {
		fmt.Printf("Failed in saving transaction to file %s, error %s\n", filename, err)
		return
	}
	fmt.Printf("Saved signed transaction to file: %s\n", filename)
}

```

## Submit the transaction from a file

This example takes the output from the previous example (file containing signed transaction) and submits it to Algod process of a node.

```golang
package main

import (
	"fmt"
	"io/ioutil"

	"github.com/algorand/go-algorand-sdk/client/v2/algod"
)

// CHANGE ME
var (
	algodAddress = "http://localhost:4001"
	algodToken   = strings.Repeat("a", 64)
)

func main() {

	rawTx, err := ioutil.ReadFile("./signed.tx")
	if err != nil {
		fmt.Printf("failed to open signed transaction: %s\n", err)
		return
	}

	// Create an algod client
	algodClient, err := algod.MakeClient(algodAddress, algodToken)
	if err != nil {
		fmt.Printf("failed to make algod client: %s\n", err)
		return
	}

	// Broadcast the transaction to the network
	sendResponse, err := algodClient.SendRawTransaction(rawTx).Do(context.Background())
	if err != nil {
		fmt.Printf("failed to send transaction: %s\n", err)
		return
	}

	fmt.Printf("Transaction ID: %s\n", sendResponse.TxID)
}
```

## Manipulating multisig transactions

Here, we first create a simple multisig payment transaction,
with three public identities and a threshold of 2:

```golang
addr1, _ := types.DecodeAddress("DN7MBMCL5JQ3PFUQS7TMX5AH4EEKOBJVDUF4TCV6WERATKFLQF4MQUPZTA")
addr2, _ := types.DecodeAddress("BFRTECKTOOE7A5LHCF3TTEOH2A7BW46IYT2SX5VP6ANKEXHZYJY77SJTVM")
addr3, _ := types.DecodeAddress("47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU")
ma, err := crypto.MultisigAccountWithParams(1, 2, []types.Address{
	addr1,
	addr2,
	addr3,
})
if err != nil {
	panic("invalid multisig parameters")
}
fromAddr, _ := ma.Address()

params, err := algodClient.SuggestedParams().Do(context.Background())
if err != nil {
	fmt.Printf("error getting suggested tx params: %s\n", err)
	return
}
txn, err := future.MakePaymentTxn(
	fromAddr.String(),
	"INSERTTOADDRESHERE",
	10000,  // amount
	nil,	// note
	"",	 // closeRemainderTo
	params,
)
txid, txBytes, err := crypto.SignMultisigTransaction(secretKey, ma, txn)
if err != nil {
	panic("could not sign multisig transaction")
}
fmt.Printf("Made partially-signed multisig transaction with TxID %s: %x\n", txid, txBytes)

```

Now, we can write the returned bytes to disk:

```golang
_ := ioutil.WriteFile("./arbitrary_file.tx", txBytes, 0644)
```

And read them back in:

```golang
readTxBytes, _ := ioutil.ReadFile("./arbitrary_file.tx")
```

Now, we can append another signature, to hit the threshold. Note that
this SDK forces new signers to know the parameters of the multisig -
after all, we don't want to sign things without knowing the identity
of the multi-signature.

```golang
// as before
addr1, _ := types.DecodeAddress("DN7MBMCL5JQ3PFUQS7TMX5AH4EEKOBJVDUF4TCV6WERATKFLQF4MQUPZTA")
addr2, _ := types.DecodeAddress("BFRTECKTOOE7A5LHCF3TTEOH2A7BW46IYT2SX5VP6ANKEXHZYJY77SJTVM")
addr3, _ := types.DecodeAddress("47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU")
ma, _ := crypto.MultisigAccountWithParams(1, 2, []types.Address{
	addr1,
	addr2,
	addr3,
})
// append our signature to readTxBytes
txid, twoOfThreeTxBytes, err := crypto.AppendMultisigTransaction(secretKey, ma, readTxBytes)
if err != nil {
	panic("could not append signature to multisig transaction")
}
fmt.Printf("Made 2-out-of-3 multisig transaction with TxID %s: %x\n", txid, twoOfThreeTxBytes)

```

We can also merge raw, partially-signed multisig transactions:

```golang
otherTxBytes := ... // generate another raw multisig transaction somehow
txid, mergedTxBytes, err := crypto.MergeMultisigTransactions(twoOfThreeTxBytes, otherTxBytes)
```

## Working with transactions group

Example below demonstrates how to create a transactions group and send it to network.

```golang
package main

import (
	"context"
	"fmt"
	"strings"

	"github.com/algorand/go-algorand-sdk/client/v2/algod"
	"github.com/algorand/go-algorand-sdk/crypto"
	"github.com/algorand/go-algorand-sdk/future"
	"github.com/algorand/go-algorand-sdk/types"
)

// CHANGE ME
var (
	algodAddress = "http://localhost:4001"
	algodToken   = strings.Repeat("a", 64)
)

func submitGroup() {

	algodClient, err := algod.MakeClient(algodAddress, algodToken)
	if err != nil {
		fmt.Printf("failed to make algod client: %v\n", err)
		return
	}

	params, err := algodClient.SuggestedParams().Do(context.Background())
	if err != nil {
		fmt.Printf("error getting suggested tx params: %s\n", err)
		return
	}

	account1 := crypto.GenerateAccount()
	fmt.Printf("account address: %s\n", account1.Address)

	account2 := crypto.GenerateAccount()
	fmt.Printf("account address: %s\n", account2.Address)

	var (
		address1 = account1.Address.String()
		address2 = account2.Address.String()
		address3 = "47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU"

		amount2 = 1500
		amount1 = 2000
		note []byte
	)

	tx1, err := future.MakePaymentTxn(address1, address2, amount1, note, "", params)
	if err != nil {
		fmt.Printf("Failed to create payment transaction: %v\n", err)
		return
	}

	tx2, err := future.MakePaymentTxn(address2, address3, amount2, note, "", params)
	if err != nil {
		fmt.Printf("Failed to create payment transaction: %v\n", err)
		return
	}

	// compute group id and put it into each transaction
	gid, err := crypto.ComputeGroupID([]types.Transaction{tx1, tx2})
	tx1.Group = gid
	tx2.Group = gid

	_, stx1, err := crypto.SignTransaction(account1.PrivateKey, tx1)
	if err != nil {
		fmt.Printf("Failed to sign transaction: %s\n", err)
		return
	}
	_, stx2, err := crypto.SignTransaction(account2.PrivateKey, tx2)
	if err != nil {
		fmt.Printf("Failed to sign transaction: %s\n", err)
		return
	}

	var signedGroup []byte
	signedGroup = append(signedGroup, stx1...)
	signedGroup = append(signedGroup, stx2...)
	_, err = algodClient.SendRawTransaction(signedGroup).Do(context.Background())
	if err != nil {
		fmt.Printf("Failed to create payment transaction: %v\n", err)
		return
	}
}

```

## Working with LogicSig

Example creates a delegating LogicSig signature signed by a MultiSig account.
A program is "int 0" that is evaluates to `FALSE` and does not actually permits the transaction.

```golang
package main

import (
	"context"
	"fmt"
	"strings"

	"github.com/algorand/go-algorand-sdk/client/v2/algod"
	"github.com/algorand/go-algorand-sdk/crypto"
	"github.com/algorand/go-algorand-sdk/future"
	"github.com/algorand/go-algorand-sdk/mnemonic"
	"github.com/algorand/go-algorand-sdk/types"
)

// CHANGE ME
var (
	algodAddress = "http://localhost:4001"
	algodToken   = strings.Repeat("a", 64)
)

func main() {
	// ignore error checking for readability
	algodClient, _ := algod.MakeClient(algodAddress, algodToken)

	addr1, _ := types.DecodeAddress("DN7MBMCL5JQ3PFUQS7TMX5AH4EEKOBJVDUF4TCV6WERATKFLQF4MQUPZTA")
	addr2, _ := types.DecodeAddress("BFRTECKTOOE7A5LHCF3TTEOH2A7BW46IYT2SX5VP6ANKEXHZYJY77SJTVM")

	mn1 := "auction inquiry lava second expand liberty glass involve ginger illness length room item discover ahead table doctor term tackle cement bonus profit right above catch"
	sk1, _ := mnemonic.ToPrivateKey(mn1)

	mn2 := "since during average anxiety protect cherry club long lawsuit loan expand embark forum theory winter park twenty ball kangaroo cram burst board host ability left"
	sk2, _ := mnemonic.ToPrivateKey(mn2)

	ma, _ := crypto.MultisigAccountWithParams(1, 2, []types.Address{
		addr1,
		addr2,
	})

	program := []byte{1, 32, 1, 0, 34} // int 0 => never transfer money
	var args [][]byte
	lsig, _ := crypto.MakeLogicSig(program, args, sk1, ma)
	_ = crypto.AppendMultisigToLogicSig(&lsig, sk2)

	sender, _ := ma.Address()
	_ = crypto.VerifyLogicSig(lsig, sender)

	var (
	 	receiver = "47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU"
	 	amount 	 = 2000

		note []byte
	)
	// Get the suggested transaction parameters
	params, _ := algodClient.SuggestedParams().Do(context.Background())

	tx, _ := future.MakePaymentTxn(sender.String(), receiver, amount, note, "", params)

	txid, stx, err := crypto.SignLogicsigTransaction(lsig, tx)
	if err != nil {
		fmt.Printf("Signing failed with %v", err)
		return
	}
	fmt.Printf("Signed tx: %v\n", txid)


	_, err = algodClient.SendRawTransaction(stx).Do(context.Background())
	if err != nil {
		fmt.Printf("Sending failed with %v\n", err)
	}
}
```

## Assets

The Algorand protocol allows users to create and trade named assets on layer one. Creating and managing these assets
is done through the issuing of asset transactions. This section details how to make asset transactions, and what they do.

Asset creation: This allows a user to issue a new asset. The user can define the number of assets in circulation,
whether there is an account that can revoke assets, whether there is an account that can freeze user accounts,
whether there is an account that can be considered the asset reserve, and whether there is an account that can change
the other accounts. The creating user can also do things like specify a name for the asset.

```golang
addr := "BH55E5RMBD4GYWXGX5W5PJ5JAHPGM5OXKDQH5DC4O2MGI7NW4H6VOE4CP4" // the account issuing the transaction; the asset creator
fee := types.MicroAlgos(10) // the number of microAlgos per byte to pay as a transaction fee
defaultFrozen := false // whether user accounts will need to be unfrozen before transacting
genesisHash, _ := base64.StdEncoding.DecodeString("SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI=") // hash of the genesis block of the network to be used
totalIssuance := uint64(100) // total number of this asset in circulation
decimals := uint64(1) // hint to GUIs for interpreting base unit
reserve := addr // specified address is considered the asset reserve (it has no special privileges, this is only informational)
freeze := addr // specified address can freeze or unfreeze user asset holdings
clawback := addr // specified address can revoke user asset holdings and send them to other addresses
manager := addr // specified address can change reserve, freeze, clawback, and manager
unitName := "tst" // used to display asset units to user
assetName := "testcoin" // "friendly name" of asset
genesisID := "" // like genesisHash this is used to specify network to be used
firstRound := types.Round(322575) // first Algorand round on which this transaction is valid
lastRound := types.Round(322575) // last Algorand round on which this transaction is valid
note := nil // arbitrary data to be stored in the transaction; here, none is stored
assetURL := "http://someurl" // optional string pointing to a URL relating to the asset
assetMetadataHash := "thisIsSomeLength32HashCommitment" // optional hash commitment of some sort relating to the asset. 32 character length.

// Get the suggested transaction parameters
params, _ := algodClient.SuggestedParams().Do(context.Background())

// signing and sending "txn" allows "addr" to create an asset
txn, err = future.MakeAssetCreateTxn(addr, note, params,
	totalIssuance, decimals, defaultFrozen, manager, reserve, freeze, clawback,
	unitName, assetName, assetURL, assetMetadataHash)
```

Asset reconfiguration: This allows the address specified as `manager` to change any of the special addresses for the asset,
such as the reserve address. To keep an address the same, it must be re-specified in each new configuration transaction.
Supplying an empty address is the same as turning the associated feature off for this asset. Once a special address
is set to the empty address, it can never change again. For example, if an asset configuration transaction specifying
`clawback=""` were issued, the associated asset could never be revoked from asset holders, and `clawback=""` would be
true for all time. The `strictEmptyAddressChecking` argument can help guard against this, it causes
`MakeAssetConfigTxn` return error if any management address is set to empty.

```golang
addr := "BH55E5RMBD4GYWXGX5W5PJ5JAHPGM5OXKDQH5DC4O2MGI7NW4H6VOE4CP4"
fee := types.MicroAlgos(10)
firstRound := types.Round(322575)
lastRound := types.Round(322575)
note := nil
genesisID := ""
genesisHash, _ := base64.StdEncoding.DecodeString("SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI=")
assetIndex := uint64(1234)
reserve := addr
freeze := addr
clawback := addr
manager := addr
strictEmptyAddressChecking := true

// Get the suggested transaction parameters
params, _ := algodClient.SuggestedParams().Do(context.Background())

// signing and sending "txn" will allow the asset manager to change:
// asset manager, asset reserve, asset freeze manager, asset revocation manager
txn, err = future.MakeAssetConfigTxn(addr, note, params,
	assetIndex, manager, reserve, freeze, clawback, strictEmptyAddressChecking)
```

Asset destruction: This allows the creator to remove the asset from the ledger, if all outstanding assets are held
by the creator.

```golang
addr := "BH55E5RMBD4GYWXGX5W5PJ5JAHPGM5OXKDQH5DC4O2MGI7NW4H6VOE4CP4"
fee := types.MicroAlgos(10)
firstRound := types.Round(322575)
lastRound := types.Round(322575)
note := nil
genesisID := ""
genesisHash, _ := base64.StdEncoding.DecodeString("SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI=")
assetIndex := uint64(1234)

params := types.SuggestedParams {
	Fee: fee,
	FirstRoundValid: firstRound,
	LastRoundValid: lastRound,
	GenesisHash: genesisHash,
	GenesisID: genesisID,
}
// if all outstanding assets are held by the asset creator,
// the asset creator can sign and issue "txn" to remove the asset from the ledger.
txn, err = MakeAssetDestroyTxn(addr, note, params, assetIndex)
```

Begin accepting an asset: Before a user can begin transacting with an asset, the user must first issue an asset acceptance transaction.
This is a special case of the asset transfer transaction, where the user sends 0 assets to themself. After issuing this transaction,
the user can begin transacting with the asset. Each new accepted asset increases the user's minimum balance.

```golang
addr := "BH55E5RMBD4GYWXGX5W5PJ5JAHPGM5OXKDQH5DC4O2MGI7NW4H6VOE4CP4"
fee := types.MicroAlgos(10)
firstRound := types.Round(322575)
lastRound := types.Round(322575)
note := nil
genesisID := ""
genesisHash, _ := base64.StdEncoding.DecodeString("SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI=")
assetIndex := uint64(1234)

params := types.SuggestedParams {
	Fee: fee,
	FirstRoundValid: firstRound,
	LastRoundValid: lastRound,
	GenesisHash: genesisHash,
	GenesisID: genesisID,
}
// signing and sending "txn" allows sender to begin accepting asset specified by creator and index
txn, err = MakeAssetAcceptanceTxn(addr, note, params, assetIndex)
```

Transfer an asset: This allows users to transact with assets, after they have issued asset acceptance transactions. The
optional `closeRemainderTo` argument can be used to stop transacting with a particular asset. Note: A frozen account can always close
out to the asset creator.

```golang
addr := "BH55E5RMBD4GYWXGX5W5PJ5JAHPGM5OXKDQH5DC4O2MGI7NW4H6VOE4CP4"
sender := addr
recipient := "47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU"
closeRemainderTo := "" // supply an address to close remaining balance after transfer to supplied address
fee := types.MicroAlgos(10)
firstRound := types.Round(322575)
lastRound := types.Round(322575)
note := nil
genesisID := ""
genesisHash, _ := base64.StdEncoding.DecodeString("SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI=")
assetIndex := uint64(1234)
amount := uint64(10)

params := types.SuggestedParams {
	Fee: fee,
	FirstRoundValid: firstRound,
	LastRoundValid: lastRound,
	GenesisHash: genesisHash,
	GenesisID: genesisID,
}

// signing and sending "txn" will send "amount" assets from "sender" to "recipient"
txn, err = MakeAssetTransferTxn(sender, recipient, amount, note, params, closeRemainderTo, assetIndex);

```

Revoke an asset: This allows an asset's revocation manager to transfer assets on behalf of another user. It will only work when
issued by the asset's revocation manager.

```golang
revocationManager := "BH55E5RMBD4GYWXGX5W5PJ5JAHPGM5OXKDQH5DC4O2MGI7NW4H6VOE4CP4" // txn signed by this address
recipient := "47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU"		 // assets sent to this address
revocationTarget := "47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU"  // assets come from this address
fee := types.MicroAlgos(10)
firstRound := types.Round(322575)
lastRound := types.Round(322575)
note := nil
genesisID := ""
genesisHash, _ := base64.StdEncoding.DecodeString("SGO1GKSzyE7IEPItTxCByw9x8FmnrCDexi9/cOUJOiI=")
assetIndex := uint64(1234)
amount := uint64(10)

params := types.SuggestedParams {
	Fee: fee,
	FirstRoundValid: firstRound,
	LastRoundValid: lastRound,
	GenesisHash: genesisHash,
	GenesisID: genesisID,
}

// signing and sending "txn" will send "amount" assets from "revocationTarget" to "recipient",
// if and only if sender == clawback manager for this asset
txn, err = MakeAssetRevocationTxn(revocationManager, revocationTarget, amount, recipient, note, params, assetIndex);
```

## Rekeying

To rekey an account to a new address, simply call the `Rekey` function on any transaction.

```golang
...
tx, err := future.MakePaymentTxn(
		account.Address.String(), "4MYUHDWHWXAKA5KA7U5PEN646VYUANBFXVJNONBK3TIMHEMWMD4UBOJBI4",
		amount, nil, "", params
	)
// From now, every transaction needs to be signed by the SK of the following address
tx.Rekey("47YPQTIGQEO7T4Y4RWDYWEKV6RTR2UNBQXBABEEGM72ESWDQNCQ52OPASU")
...
```

When submitting a transaction from an account that was rekeying, simply use relevant SK. `SignTransaction` will detect
that the SK corresponding address is different than the sender's and will set the `AuthAddr` accordingly.
