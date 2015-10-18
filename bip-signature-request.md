==Abstract==

This BIP outlines a method for remote parties to request a signature for a 
specific unspent transaction output. By standardizing requests for signatures, 
this scheme enables multi-signature transactions to be conducted between users 
each using their own devices and software. In essence, while BIP70 challenges 
a user to sign any inputs which meet the specified outputs, this proposal 
challenges a user to produce add their signatures for a specific set of UTXO's. 

==Motivation==

To date most multi-signature wallets force users to depend on software they 
provide, instead of the users own. This amounts to a vendor lockin of sorts, 
not to mention introducing a custodial relationship. It also exposes users to 
the risk of leaks to their private key due to faulty implementations – as has 
happened Blockchain.info twice this year. 

In addition to requesting signatures for a multisignature transaction, the 
Lightning network also requires some means of requesting signatures. 

The intention of this BIP is to allow a distributed group of users to sign a 
transaction, each using a device or wallet application of their chosing, and
to make fulfilling these requests easier in general.

Also, there is merit to having a standard for (i) requesting (extended) public 
keys, (ii) informing a user of a new script, (iii) requesting payment to a 
particular output, and (iv) requesting a signature for a specific utxo. 

==Rationale==
The Output message taken as-is from BIP70 fulfills our requirements without change. 

The UTXO message was designed such that everything required for signing must be 
provided, including the output script of the transaction being spent. This may 
enable some situations where wallets can't arbitrary lookup outputs by the script.

The user must be provided with the transaction version, locktime, all outputs, all 
inputs and corresponding output scripts if signing is required), if a signature is 
to the calculated entirely on the device. This makes the requests quite large. Some 
fields could be removed if the signature hash was included, however this requires 
too much trust in the party constructing it.

The wallet application will somehow have to verify the service is permitted to 
request a signature for this utxo. Open question for now. 

==Protocol==
Suppose a people have have exchanged a multi-signature address and script, or 
initiated a Lightning network payment channel. Funds have been sent to the address, 
and now a signature is required from one of the parties. 

The request can be initiated through a bitcoin URI, NFC, or a QR code. 

(1) Following some action, the users wallet visits the unique request URI.
(2) The server constructs the Output and Utxo messages, the SignatureDetails and SignatureRequest messages, and sends the request to the client.
(3) The wallet application checks if the utxos correspond to known inputs. The scriptPubKey should only be set for utxos which are to be signed for the user. The user is prompted to sign the transaction.
(4) If the user accepts, SignatureScript messages are created which fulfill as much of the requirements as possible. This means adding j signatures if they are all found on their device. They are compiled into a SignatureResponse message. Should the utxo messages scriptPubKey value be empty, the SignatureScript's scriptSig value will automatically be empty. The SignatureResponse is sent to the server.
(5) The server verifies the signatures and ensures all requirements are met. If so, a SignatureAck message is sent to the user. 

==Messages==

===Utxo===
<pre>
    message Utxo {
        required bytes txid = 1;
        required uint32 vout = 2;
        optional uint32 sequence = 3 [default = 0xffffffff];
        required bytes scriptPubKey = 4;
    }
</pre>

{|
| TXID        || Transaction ID of the output to be spent 
|-
| Vout        || Index of the output in the specified transaction
|-
| Sequence    || Sequence number for this output
|-
| ScriptPubKey|| ScriptPubKey of this output, which is required if the user is to create the transaction. If this is P2SH, the script will need to be on the device. 
|}

===Output===

Taken as-is from BIP70. See [[bip-0070.mediawiki#output|BIP70: Output]]

===SignatureDetails===

<pre>
    message SignatureDetails {
        repeated Utxo utxos = 1;
        repeated Output outputs = 2;
        required uint32 txver = 3;
        optional uint32 nLockTime = 4 [default = ....];
        required uint64 time = 5;
        optional uint64 expires = 6;
        optional string memo = 7;
        optional string signature_url = 8;
        optional bytes merchant_data = 9;
    }
</pre>

{|
| Utxos        || Transaction ID of the output to be spent 
|-
| Outputs        || Index of the output in the specified transaction
|-
| Tx Version    || Sequence number for this output
|-
| nLockTime || ScriptPubKey of this output, which is required if the user is to create the transaction. If this is P2SH, the script will need to be on the device. 
|-
| Time || 
|-
| Expires || 
|-
| Memo || 
|-
| Signature URL || 
|-
| Merchant Data || 
|}

<pre>
    message SignatureRequest {
        optional uint32 signature_request_version = 1 [default: 1];
        optional string pki_type = 2 [default = “none”];
        optional bytes pki_data = 3;
        required bytes serialized_signature_details = 4;
        optional bytes signature = 5;
    }
</pre>


{|
| Signature Request Version ||  
|-
| PKI Type  || 
|-
| PKI Data  ||
|-
| Serialized Signature Details ||  
|-
| Signature ||  
|}

<pre>
    message SignatureScript {
        required bytes scriptSig;
    }
</pre>

{|
| ScriptSig || Generated scriptSig, containing as many signatures as possible. Or more generally, a scriptSig their device produces, solves the scriptPubKey to the best of their ability.
|-
|}

<pre>
    message SignatureResponse {
        repeated SignatureScript scripts = 1;
        optional string merchant_data = 2;
        optional string memo = 3; 
    }
</pre>

{|
| ScriptSigs ||  
|-
| Merchant Data  || 
|-
| Memo ||  
|}

