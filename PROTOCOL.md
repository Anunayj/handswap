# Protocol

To understand how cross chain atomic swaps generally work, refer to [[1]](#ref), the protocol implemented here is slight variation of it, and this document is for documentation purposes.


## Communication
Initial communication and price negotionation is done off site. Once this had been done, the browsers of two parties may communicate over WebRTC (using something like [PeerJS](https://peerjs.com/)) or other server assisted protocols.

## Protocol Details
Assume Alice has a balance of 100 HNS and wants to swap it for 1 BTC with Bob.

### Step -1
Both Alice and Bob generate a mnemonic, this will be used for deriving public and private keys used for the swap. This can also be optionally stored in browser localstorage (encrypted?), or generated on each visit. Unfortunately Web Crypto API cannot be used for this because it doesn't support secp256k1 yet, (and won't likely in future  [[2]](#ref).

This mnemonic (combined with some other information) will be needed to refund funds in case something goes wrong.

### Step 0
Alice fills in trade parameters (Amount and Exchange rate), and generates a link/id to send over to Bob, this id will let Bob connect to Alice over WebRTC (using PeerJS). Bob can choose to reject the trade if he doesn't agree to the trade parameters.

### Step 1
Alice creates the following contract:
```
OP_IF
  OP_2 <pubKeyC> <pubKeyA> OP_2 OP_CHECKMULTISIG
OP_ELSE
  <penalty locktime>
  OP_CHECKSEQUENCEVERIFY/OP_CHECKLOCKTIMEVERIFY
  OP_DROP
  <pubKeyA>
  OP_CHECKSIG
OP_ENDIF
```
Where pubKeyC is the public key of a trusted third party (C),
This allows the output to be spent by two paths:
1. From a multisignature from both C and A.
2. By A alone after the locktime has passed.

Alice then sends her 100 HNS to this address, and sends the tx (with the redeem script) to Bob to verify, Bob waits for 1 confirmation.

This step is *optional* (but recommended) to combat liquidity trolling.

### Step 2





## References:
<a id="ref"></a> 

1. https://bcoin.io/guides/swaps
2. https://github.com/w3c/webcrypto/issues/82
