<pre>
  BIP: pubkey-request
  Title: Public Key Request Protocol
  Authors: Thomas Kerin
  Status: Final
  Type: Standards Track
  Created: 2015-05-30
</pre>

==Abstract==
This BIP describes a protocol to facilitate requests for public keys, and extended public keys from a users wallet.
 
==Motivation==
 Since the adoption of multi-signature wallets by the community, there have been no 
 developments enabling the request of a public key by a server or remote user.
 
 Because of this, there have been very few services which allow clients to manage
 their own wallet, and issue signatures using their own software. Instead, 
 all users of such a service all use the same code. 
      
 The majority of cases which do accept regular or extended public keys require 
 users to enter them manually, making the process error prone. 
   
 This BIP attempts to address this by providing a simple means to request a 
 public key from a user, or an extended public key from compatible wallets using 
 BIP70 like requests.
        
 It is complementary to BIP70, as following request of a public key the user may 
 now have to fund a contract involving that key. 
 
==Protocol==

The protocol largely follows BIP70 in it's use of Google's Protocol Buffers, where the sender of the request may opt to authenticate via X.509 certificates. 

==Messages==

===KeyDetails/KeyRequest===

<pre>
    message KeyDetails {
      optional string type = 1 [default = “standard”]
      required uint64 time = 2;
      optional uint64 expires = 3;
      optional string memo = 4;
      required string key_url = 5;
      optional bytes merchant = 6;
    }
</pre>

{|
| Type || Either "standard" for a regular public key, or "extended" for BIP32 extended public key. 
|-
| Time || Unix timestamp (seconds since 1-Jan-1970 UTC) when the KeyRequest was created.
|-
| Expires || Unix timestamp (UTC) after which the request should be considered invalid.
|-
| Memo || UTF-8 encoded, plain-text (no formatting) note to display to the recipient of the request.
|- 
| Key URL || Secure (https) location where the Key message (see below) must be sent.
|-
| Merchant Data || Arbitrary bytes specified by the sender to identify the KeyRequest.
|}


The KeyRequest may, and should be tied to the services identity. Authentication via X.509 comes in 
the form of a certificate chain, specifying the pki_type, and providing a signature for the serialized 
KeyRequest. This message is identical to PaymentRequest of BIP70 (save the serialized_key_details field).

<pre>
    message KeyRequest {
        optional uint32 key_details_version = 1 [default = 1];
        optional string pki_type = 2 [default = “none”];
        optional bytes pki_data = 3;
        required bytes serialized_key_details = 4;
        optional bytes signature = 5;
    }
</pre>

{|
| Key Details Version || See below for discussions on upgrading 
|-
| PKI Type || Public Key Infrastructure system being used to identify the merchant. All implementations should accept the types “none”, “x506+sha256”, and “x509+sha1” to maintain compatibility with BIP70.
|-
| PKI Data || PKI system data that identifies the merchant, typically one or more X.509 certificates. (See Certificates message below)
|-
| Serialized Key Details || A protocol-buffer serialized KeyDetails message
|- 
| Signature || A digital signature of the hash of the serialized key details
|}

 Upon receipt of a KeyRequest, a wallet must assess whether it can fulfill the 
 requested key type, and that the key_details_version is supported. If it 
 cannot fulfill the request due to compatibility, it should send a KeyReject 
 message, and advise the user of this failure.  

 The ability to request a standard public key exists to allow once off payments,
 or should an extended key be unavailable and the service is willing to accept 
 a single public key in a new KeyRequest.
 
===Key===
<pre>
    message Key {
        optional bytes merchant_data = 1;
        required key_data = 2;
        optional string memo = 3;
    }
</pre>

 The Key message is sent to fulfill the servers request. It should provide a key
 of the same type as was requested, otherwise the request should fail. 
 
 If a standard key is requested, the wallet should derive a new standard bitcoin 
 private key to be kept locally, and respond with the public key. Since standard 
 public key requests may be for a once off encounter, if the wallet application 
 supports BIP32 it should derive the next available key a general purpose chain. 
 
 If an extended key is requested, the wallet application should perform a new 
 child key derivation (CKD), yielding a child key which is specific to the service. 

 Regular public keys are sent in DER representation, and extended public keys should
 be sent as their binary serialized representation, and not as base58. 
 

===KeyReject===
<pre>
    message Reject {
        optional bytes merchant_data = 1;
    }
</pre>

 A KeyReject message is sent should the user decline the offer to share a public 
 key with the service, or if the application is unable to support the requested 
 key type. 

 While this is not a strict requirement, it may help to inform the service of 
 failure to allow for a fallback to a standard public key if possible. Note: 
 this is not automatic, and would involve a new KeyRequest.
 
===KeyACK===
<pre>
    message KeyAck {
        required Key key = 1;
        optional string memo = 2;
    }
</pre>

A KeyACK message is sent upon successful receipt of a Key message. 

===Certificates===

 As defined in BIP70, Certificate protocol buffers are used to include a a chain
 of certificates which authenticate a provided signature. 

 To avoid duplication, for details + this message structure, see [[bip-0070.mediawiki#certificates|BIP 70 X509 Certificates]] 

==Extensibility==
 The KeyRequest message contains a key_details_version parameter. The only option
 at present is version 1, which permits 'extended' or 'standard' public keys to 
 be requested. Future desirable schemes can be introduced in a with a new version. 

 Upgrades with extra fields, which still remain “version 1” will pass with a valid
 signature, however the fields will remain unavailable. 

==References==
[[bip-0070.mediawiki|BIP 0070 - Payment Protocol]]
[[https://developers.google.com/protocol-buffers/|Protocol Buffers]]
[[http://datatracker.ietf.org/wg/pkix/charter|PKIX Working Group]]