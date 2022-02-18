# Protocol

To understand how cross chain atomic swaps generally work, refer to [this](https://bcoin.io/guides/swaps), the protocol implemented here is slight variation of it, and this document is for documentation purposes.


## Communication
Initial communication and price negotionation is done off site. Once this had been done, the browsers of two parties may communicate over WebRTC (using something like [PeerJS](https://peerjs.com/)) or other server assisted protocols.

## Protocol Details
Assume Alice has a balance of 100 HNS and wants to swap it for 1 BTC with Bob.

Step -1:
Both Alice and Bob generate a mnemonic, this will be used for deriving public and private keys used for the swap. This can also be optionally stored in browser localstorage (encrypted?), or generated on each visit. Unfortunately Web Crypto API cannot be used for this because it doesn't support secp256k1 yet, (and won't likely in future [2]).

This mnemonic (combined with some other information) will be needed to refund funds in case something goes wrong.

Step 0: 







References:

1. https://bcoin.io/guides/swaps
2. https://github.com/w3c/webcrypto/issues/82