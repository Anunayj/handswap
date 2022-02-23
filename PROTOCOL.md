# Protocol

To understand how cross chain atomic swaps generally work, refer to [[1]](#ref), the protocol implemented here is a slight variation of it, and this document is for documentation purposes.


## Communication
Initial communication and price negotiation are done off-site. Once this had been done, the browsers of two parties may communicate over WebRTC (using something like [PeerJS](https://peerjs.com/)) or other server-assisted protocols.

## Protocol Details
Assume Alice has a balance of 100 HNS and wants to swap it for 1 BTC with Bob.

### Step -1
Both Alice and Bob generate a mnemonic, this will be used for deriving public and private keys used for the swap. This can also be optionally stored in browser localstorage (encrypted?), or generated on each visit. Unfortunately, Web Crypto API cannot be used for this because it doesn't support secp256k1 yet, (and won't likely in the future  [[2]](#ref)).

This mnemonic (combined with some other information) will be needed to refund funds in case something goes wrong.

### Step 0
Alice fills in trade parameters (Amount and Exchange rate), and generates a link/id to send over to Bob, this id will let Bob connect to Alice over WebRTC (using PeerJS). Bob can choose to reject the trade if he doesn't agree with the trade parameters.

### Step 1
Alice creates the following contract (on HNS):
```
OP_IF
  OP_2 <pubKeyC> <pubKeyA> OP_2 OP_CHECKMULTISIG
OP_ELSE
  <penalty locktime> OP_CHECKSEQUENCEVERIFY OP_DROP
  <pubKeyA> OP_CHECKSIG
OP_ENDIF
```
Where pubKeyC is the public key of a trusted third party (C),
This allows the output to be spent by two paths:
1. From a multisignature from both C and A.
2. By A alone after the locktime has passed.

Alice then sends > 100 HNS + (predicted fee) to this address. Carol signals to Bob this step has been done. (One can attempt to make this more trustless, but it doesn't matter)

This step is *optional* (but recommended) to combat liquidity trolling.

### Step 2
Bob asks Alice for her public key.
Bob generates a 32-byte nonce and sends its SHA-256 and the following contract (on BTC) to Alice to verify, along with his public key.
```
OP_IF
  OP_SHA256 <hash of nonce> OP_EQUALVERIFY
  <pubKeyA> OP_CHECKSIG
OP_ELSE
  <locktime1> OP_CHECKSEQUENCEVERIFY OP_DROP
  <pubKeyB> OP_CHECKSIG
OP_ENDIF
```
Bob also sends 1 BTC to this address and broadcasts the transaction. Ideally opting out of RBF.
### Step 3
Alice creates the following contract (on HNS):
```
OP_IF
  OP_SIZE 32 OP_EQUALVERIFY 
  OP_SHA256 <hash of nonce> OP_EQUALVERIFY
  OP_2 <pubkeyA> <pubKeyB> OP_2 OP_CHECKMULTISIG
OP_ELSE
  <locktime2> OP_CHECKSEQUENCEVERIFY OP_DROP
  <pubKeyA> OP_CHECKSIG
OP_ENDIF
```
Where locktime1 < locktime2 (to ensure Alice has time to claim BTC after the nonce is revealed)
Alice also sends her 100 HNS to this address. (After awaiting a appropriate amount of time, if the other person opted out of RBF - In mempool, else till 1 confirmation) (With the signature from the trusted third party, she can keep any excess change)

A multisig is used here to allow Alice not to have to wait 3 blocks for Bob's transaction to confirm.
### Step 4
Alice and Bob wait 3 confirmations (for both their transfers to confirm). Bob then creates a transaction transferring the 100 HNS on his address (but does not sign/reveal the nonce), and asks for Alice's signature. On recieving the signature, he verifies it, and then spends it revealing the nonce.

Alice monitors the chain for a transaction revealing the nonce and claims 1 BTC.
(Alice can also presign a transaction and store it in a watchtower to do it for her)

Bob should now have 100 HNS - 1x (HNS tx fee) in his wallet.
Alice should have  1 BTC - 1x (BTC TX fee) in his wallet.

### Timelock length
Blockheight Based timelocks are not a good option because the probablistic nature of mining can cause two chains with similar target block time to diverge.
So timestamp based must be timelocks are used.
The following timelocks are used:

- locktime1 = 6 hour (relative) 
- locktime2 = 12 hours (relative)
- penalty timelock = 48 hours (relative)
 
## Common Limitations and Security issues.

### Liquidity Trolling

If step 1 is skipped, there is no incentive for Alice to stay after Step 2, which would mean Bob's funds are effectively locked for <locktime> duration at no expense to Alice, having a trusted 3rd party that both users trust can allow us to punish Alice if she leaves after step 2.
Alice's funds can be reversed to her in case Bob leaves after step 2, at the cost of 2x TX fee to her.

### Nonce size attack
On some chains length of nonce can be selected by Bob in such a way that it is not possible to claim it in one of the chains, while is the other way around [[3]](#ref). This isn't strictly a problem with HNS and BTC since both have a 520 byte limit element size, but the length check is still kept in for security reasons.

### Trust on centralized servers
Trust on a single centralized server must be largely avoided, this can be achieved by implementing a partial SPV like protocol (just the block headers for the past few blocks with appropriate difficulty should be enough) and Proof of Existence of tx. 

The "trusted third party" at any point doesn't control funds unilaterally and can be completely bypassed by waiting for the penalty locktime.

However, there is no ideal way to verify the Proof of Inexistance of a transaction of Bob's spend, which means if Bob and all nodes choose to lie to Alice, she may have no way to claim her 1 BTC. However she only needs 1 honest node to reveal the nonce to her.

### The Pull Out tactic (someone suggest a better name)
At step 4, Bob can choose to wait/not go through with the swap unless the market moves in his favour. It is assumed harsh locktimes would make such tactics unfeasible.

## Security Circumstances
Cryptography and Browsers don't mix well. There are many ways extensions can interfere/steal funds while the swap is taking place, however considering that the funds spend ~ 60 minutes in browser held keys, I'd argue this should be safe enough. Also, don't install random extensions :) and use the latest browser.

## Privacy
Due to the shared nonce, Cross chain atomic swaps aren't private. However, unlike centralized exchanges there is no need for a KYC. So cross chain atomic swaps are attractive for those wanting to maximize their privacy but don't really care about chain analysis.

However, it is possible to have more private atomic swaps using Adaptor Signatures, but is outside the scope of this project.

## Potential Improvements
There are a few ways this protcol can be improved 
1. Use of Adaptor signatures - Increase privacy, remove the need for a common hashing algorithm, Complicated to implement.
2. Advanced knowledge of tx_hash of funding transactions will allow reduction size of the BTC contract by 30% (By the use of a presigned refund tx). 

## References:
<a id="ref"></a> 

1. https://bcoin.io/guides/swaps
2. https://github.com/w3c/webcrypto/issues/82
3. https://gist.github.com/markblundeberg/7a932c98179de2190049f5823907c016
