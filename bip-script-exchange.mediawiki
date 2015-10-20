<pre>
  BIP: script-exchange
  Title: Script Exchange protocol
  Authors: Thomas Kerin
  Status: Final
  Type: Standards Track
  Created: 2015-05-30
</pre>

==Abstract==
 This BIP outlines a proposal for one entity to inform anothers Bitcoin wallet 
 application about scripts pertaining to a key in their wallet. 
 
==Motivation==
 Most users in bitcoin are familiar with the concepts of addresses. There are
 two types - pay-to-pubkey-hash, and pay-to-script-hash.   
 
 The model of creating a new address type for every interesting contract is not
 scalable, and hence we have seen BIP70 emerge, which explicitly defines output
 scripts for users to fund. 
 
 Whilst this is useful for receiving payment, the case where someone needs to 
 sign inputs is not yet catered for. 
  
 To address this, a means of informing a wallet about an output or pay-to-script-hash 
 script is proposed. Pay-to-script-hash is a special case, because simply knowing
 the address or output script does not enable you to sign. This would allow any 
 full-node or SPV wallet to find outputs corresponding to this output script, 
 and also allow them to sign transactions. 
  
 It is intended that wallets will support redeeming these outputs on a best-effort
 basis. For compatibility, wallets should reject output scripts they cannot interpret.
 
 The primary motivation of this BIP is to offload the responsibility of signing 
 to the clients device, instead of relying on particular software. Similarly,
 wallets may support for any output script on a best effort basis, and reject
 those they cannot interpret.  
 
 This avoids the situation of "one wallet fits all", permitting specialized 
 applications to be developed, but ultimately allowing for alternatives. 
 
==Protocol==

The protocol largely mirrors BIP70, in that details are sent to the user for approval, and can be authenticated by X.509 certificates. 

The exchange can be initiated by a bitcoin URI, a QR code, or via NFC.  

1. First the user clicks 'Pair my device' or something. 
2. The wallet application visits the script url (encoded via QR, or using 
   specialized URI). 
3. The server and prepares the Script / ScriptDetails / ScriptRequest, and 
   sends the message to the client.
4. The client parses the message, checking signatures if available. 
   (i)  If it can't support the script, it must inform the user, and 
        send a ScriptReject. 
   (ii) If the script is supported, the application should prompt the 
        user to accept the script, displaying all the available details. 
5. If the user accepts the script, a ScriptACK message is sent to the server. 
   The wallet application should then search for outputs encumbered with this script.
(6?). The server responds with the ScriptACK to confirm. 

==Messages==

===Script/ScriptRequest===
<pre>
    message ScriptDetails {
        required Script script = 1;
        required uint64 time = 2;
        optional uint64 expires = 3;
        optional string memo = 4;
        required string script_url = 5;
        optional bytes merchant_data = 6;
    }
</pre>

{|
| Script        || The Script message (see below) 
|-
| Time          || Unix timestamp (seconds since 1-Jan-1970 UTC) when the request was created.
|-
| Expires       || Unix timestamp (UTC) after which the request should be considered invalid.
|-
| Memo          || UTF-8 encoded, plain-text (no formatting) note to display to the recipient of the request.
|- 
| Script URL    || Secure (https) location the client can submit it's ScriptACK to.
|-
| Merchant Data || Arbitrary bytes specified by the sender to identify the request.
|}

<pre>
    message ScriptRequest {
        optional uint32 script_details_version = 1 [default = 1];
        optional string pki_type = 2 [default = “none”];
        optional bytes pki_data = 3;
        required bytes serialized_script_details = 4;
        optional bytes signature = 5;
    }
</pre>

{|
| Script Details Version || See below for discussions on upgrading 
|-
| PKI Type || Public Key Infrastructure system being used to identify the merchant. All implementations should accept the types “none”, “x506+sha256”, and “x509+sha1” to maintain compatibility with BIP70.
|-
| PKI Data || PKI system data that identifies the merchant, typically one or more X.509 certificates. (See Certificates message below)
|-
| Serialized Script Details || A protocol-buffer serialized ScriptDetails message
|- 
| Signature || A digital signature of the hash of the serialized script details
|}

The script, and the script type, are specified in ScriptDetails. Bitcoin Core 
scrapes outputs marked as spendable by comparing them to a template of each 
known script type. The same can be done here to ensure the wallet is capable 
of redeeming the funds. If the wallet is unable to service these outputs, it 
can simply fail and inform the user. 

The details of each script will vary, but each acceptable script should have
 a custom UI handler which describes the details of the script – for 
 multi-signature it would display the number of participants, the required 
 number of keys, and verify that at least one key belongs to the user. 
 
===Script===
<pre>
    message Script {
        required bytes script = 1;
        required ..... type = 2;
    };
</pre>

{|
| Script || The binary representation of the script 
|-
| Type || The script type can be either "output" or "p2sh". 
|}

 If the type is 'output', the script is used directly as the transaction output
 script. If the type is 'p2sh', the script is hashed and encoded as the P2SH 
 output script. 

 
===ScriptACK===
<pre>
    message ScriptAck {
       optional bytes merchant_data = 1;
       required Script script = 2;
       optional string memo = 3;
    }
</pre>

 The ScriptACK message is returned by the users wallet application to the server 
 if the user accepts the script. 

===ScriptReject===
<pre>
    message ScriptReject {
        optional bytes merchant_data = 1;
    }
</pre>

A KeyACK message is sent upon successful receipt of a Key message. 

===Certificates===

 As defined in BIP70, Certificate protocol buffers are used to include a a chain
 of certificates which authenticate a provided signature. 

 To avoid duplication, for details + this message structure, see [[bip-0070.mediawiki#certificates|BIP 70 X509 Certificates]] 

==Extensibility==
 The ScriptRequest message contains a script_details_version parameter. The only option
 at present is version 1, which permits 'output' or 'p2sh' script types. This should
 cover most use cases, but if specialized messages are required for a script, they
 may be implied with a new version. 

 Upgrades with extra fields, which still remain “version 1” will pass with a valid
 signature, however the fields will remain unavailable. 

==References==
[[bip-0070.mediawiki|BIP 0070 - Payment Protocol]]
[[https://developers.google.com/protocol-buffers/|Protocol Buffers]]
[[http://datatracker.ietf.org/wg/pkix/charter|PKIX Working Group]]