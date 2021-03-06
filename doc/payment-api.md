# Zcash Payment API

## Overview

Zcash extends the Bitcoin Core API with new RPC calls to support private Zcash payments.

Zcash payments make use of two address formats:

* taddr - an address for transparent funds (just like a Bitcoin address, value stored in UTXOs)
* zaddr - an address for private funds (value stored in objects called notes)

When transferring funds from one taddr to another taddr, you can use either the existing Bitcoin RPC calls or the new Zcash RPC calls.

When a transfer involves zaddrs, you must use the new Zcash RPC calls.


## Compatibility with Bitcoin Core

Zcash supports all commands in the Bitcoin Core API (as of version 0.11.2).   Where applicable, Zcash will extend commands in a backwards-compatible way to enable additional functionality.

We do not recommend use of accounts which are now deprecated in Bitcoin Core.  Where the account parameter exists in the API, please use “” as its value.

To support multiple users in a single node’s wallet, consider using getnewaddress or z_getnewaddress to obtain a new address for each user.  Also consider mapping multiple addresses to each user.

## List of Zcash API commands

Optional parameters are denoted in [square brackets].

RPC calls by category:

* Accounting: z_getbalance, z_gettotalbalance
* Addresses : z_getnewaddress, z_listaddresses
* Keys : z_exportkey, z_importkey, z_exportwallet, z_importwallet
* Operation: z_getoperationresult, z_getoperationstatus, z_listoperationids
* Payment : z_listreceivedbyaddress, z_sendmany

RPC parameter conventions:

* taddr : Transparent address
* zaddr : Private address
* address : Accepts both private and transparent addresses.
* amount : JSON format double-precision number with 1 ZC expressed as 1.00000000.
* memo : Metadata expressed in hexadecimal format.  Limited to 512 bytes, the current size of the memo field of a private transaction.  Zero padding is automatic.

### Accounting

Command | Parameters | Description
--- | --- | ---
z_getbalance<br>| address [minconf=1] | Returns the balance of a taddr or zaddr belonging to the node’s wallet.<br><br>Optionally set the minimum number of confirmations a private or transaction transaction must have in order to be included in the balance.  Use 0 to count unconfirmed transactions.
z_gettotalbalance<br>| [minconf=1] | Return the total value of funds stored in the node’s wallet.<br><br>Optionally set the minimum number of confirmations a private or transparent transaction must have in order to be included in the balance.  Use 0 to count unconfirmed transactions.<br><br>Output:<br>{<br>"transparent" : 1.23,<br>"private" : 4.56,<br>"total" : 5.79}

### Addresses

Command | Parameters | Description
--- | --- | ---
z_getnewaddress | | Return a new zaddr for sending and receiving payments. The spending key for this zaddr will be added to the node’s wallet.<br><br>Output:<br>zN68D8hSs3...
z_listaddresses | | Returns a list of all the zaddrs in this node’s wallet for which you have a spending key.<br><br>Output:<br>{ [“z123…”, “z456...”, “z789...”] }

### Key Management

Command | Parameters | Description
--- | --- | ---
z_exportkey | zaddr | _Requires an unlocked wallet or an unencrypted wallet._<br><br>Return a zkey for a given zaddr belonging to the node’s wallet.<br><br>The key will be returned as a string formatted using Base58Check as described in the Zcash protocol spec.<br><br>Output:AKWUAkypwQjhZ6LLNaMuuuLcmZ6gt5UFyo8m3jGutvALmwZKLdR5
z_importkey | zkey [rescan=true] | _Wallet must be unlocked._<br><br>Add a zkey as returned by z_exportkey to a node's wallet.<br><br>The key should be formatted using Base58Check as described in the Zcash protocol spec.<br><br>Set rescan to true (the default) to rescan the entire local block database for transactions affecting any address or pubkey script in the wallet (including transactions affecting the newly-added address for this spending key).<br><br>**NOTE:** If funds for imported keys fail to display, try stopping the node and restarting it with a `-reindex` flag. (`./src/zcashd -reindex`)
z_exportwallet | filename | _Requires an unlocked wallet or an unencrypted wallet._<br><br>Creates or overwrites a file with taddr private keys and zaddr private keys in a human-readable format.<br><br>Filename is the file in which the wallet dump will be placed. May be prefaced by an absolute file path. An existing file with that name will be overwritten.<br><br>No value is returned but a JSON-RPC error will be reported if a failure occurred.
z_importwallet | filename | _Requires an unlocked wallet or an unencrypted wallet._<br><br>Imports private keys from a file in wallet export file format (see z_exportwallet). These keys will be added to the keys currently in the wallet. This call may need to rescan all or parts of the block chain for transactions affecting the newly-added keys, which may take several minutes.<br><br>Filename is the file to import. The path is relative to zcashd’s working directory.<br><br>No value is returned but a JSON-RPC error will be reported if a failure occurred.

### Payment

Command | Parameters | Description
--- | --- | ---
z_listreceivedbyaddress<br> | zaddr [minconf=1] | Return a list of amounts received by a zaddr belonging to the node’s wallet.<br><br>Optionally set the minimum number of confirmations which a received amount must have in order to be included in the result.  Use 0 to count unconfirmed transactions.<br><br>Output:<br>[{<br>“txid”: “4a0f…”,<br>“amount”: 0.54,<br>“memo”:”F0FF…”,}, {...}, {...}<br>]
z_sendmany<br> | fromaddress amounts [minconf=1] | _This is an Asynchronous RPC call_<br><br>Send funds from an address to multiple outputs.  The address can be either a taddr or a zaddr.<br><br>Amounts is a list containing key/value pairs corresponding to the addresses and amount to pay.  Each output address can be in taddr or zaddr format.<br><br>When sending to a zaddr, you also have the option of attaching a memo in hexadecimal format.<br><br>Example of Outputs parameter:<br>[{“address”:”t123…”, “amount”:0.005},<br>,{“address”:”z010…”,”amount”:0.03, “memo”:”f508af…”}]<br><br>Optionally set the minimum number of confirmations which a private or transparent transaction must have in order to be used as an input.<br><br>The transaction fee will be determined by the node’s wallet.  Any transparent change will be sent to a new transparent address.  Any private change will be sent back to the zaddr being used as the source of funds.<br><br>Returns an operationid.  You use the operationid value with z_getoperationstatus and z_getoperationresult to obtain the result of sending funds, which if successful, will be a txid.

### Operations

Asynchronous calls return an OperationStatus object which is a JSON object with the following defined key-value pairs:

* operationid : unique identifier for the async operation.  Use this value with z_getoperationstatus or z_getoperationresult to poll and query the operation and obtain its result.
* status : current status of operation
  * queued : operation is pending execution
  * executing : operation is currently being executed
  * cancelled
  * failed.
  * success
* result : result object if the status is ‘success’.  The exact form of the result object is dependent on the call itself.
* error: error object if the status is ‘failed’. The error object has the following key-value pairs:
  * code : number
  * message: error message

Currently, as soon as you retrieve the operation status for an operation which has finished, that is it has either succeeded, failed, or been cancelled, the operation and any associated information is removed.

It is currently not possible to cancel operations.

Command | Parameters | Description
--- | --- | ---
z_getoperationresult <br>| [operationids] | Return OperationStatus JSON objects for all completed operations the node is currently aware of, and then remove the operation from memory.<br><br>Operationids is an optional array to filter which operations you want to receive status objects for.<br><br>Output is a list of operation status objects, where the status is either "failed", "cancelled" or "success".<br>[<br>{“operationid”: “opid-11ee…”,<br>“status”: “cancelled”},<br>{“operationid”: “opid-9876”, “status”: ”failed”},<br>{“operationid”: “opid-0e0e”,<br>“status”:”success”,<br>“execution_time”:”25”,<br>“result”: {“txid”:”af3887654…”,...}<br>},<br>]
z_getoperationstatus <br>| [operationids] | Return OperationStatus JSON objects for all operations the node is currently aware of.<br><br>Operationids is an optional array to filter which operations you want to receive status objects for.<br><br>Output is a list of operation status objects.<br>[<br>{“operationid”: “opid-12ee…”,<br>“status”: “queued”},<br>{“operationid”: “opd-098a…”, “status”: ”executing”},<br>{“operationid”: “opid-9876”, “status”: ”failed”}<br>]<br><br>When the operation succeeds, the status object will also include the result.<br><br>{“operationid”: “opid-0e0e”,<br>“status”:”success”,<br>“execution_time”:”25”,<br>“result”: {“txid”:”af3887654…”,...}<br>}
z_listoperationids <br>| [state] | Return a list of operationids for all operations which the node is currently aware of.<br><br>State is an optional string parameter to filter the operations you want listed by their state.  Acceptable parameter values are ‘queued’, ‘executing’, ‘success’, ‘failed’, ‘cancelled’.<br><br>[“opid-0e0e…”, “opid-1af4…”, … ]
