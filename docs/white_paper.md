# LVBsig Lamport-Scheme-Based Signature for Bitcoin Layer 1

We propose a hybrid-key signature scheme consisting of a conventional signing scheme with irrepudiable signature and [Lamport scheme](https://en.wikipedia.org/wiki/S/KEY). The benefit of it is a significant reduction of onchain footprint as will be demonstrated below.

## Notation

* **H(.)** = serial-work- and memory-*hard* hash;
* **h(.)** = nonspecific representation of conventional hashes;
* **KDF(.)** = conventional key deriving function;
* **PUBi** = public key correspondent to PRI_i;
* **PRIi** = KDF(ADDR_(i-1) = conventional Bitcoin signing key;
* **ADDRi** = h(PUB_i) = i-th Bitcoin address of the chain;
* **TXi** = plaintext transaction
* **SALTi** = Contacenation of nonces of a few blocks recent by the time of issuing of TXi, say T0i-6, ... Ti-13
* **LSIGi** = h(TXi,ADDR(i-1),SALTi)
* **COMi** = "commitment" = Smart contract stating "This UTXO is frozen until one of the following happens:
  * A) publishing of an L such that h(TXi,L,SALTi) = LVBSIGi before T2 in which case TX is deemed valid and executed;
  * B) T2 blocks from now, when miner of BLi has gets additional FF0 (on top of what is established by TX), and the miner M2 of COMMITMENT gets F2, both from UTXO"; or
  * C) another such commitment is signed by the same key (attempted double spending), in which case both (or all) are mined with their respective F2's, and respective miners of (TXi,LVBSIGi)'s get their respective FF0i's (again, on top of what is established by their respective TXi's);
* **F0i** = fee offered to mine BLi;
* **FF0i** = fine offered to miner of BL to compensate for delay (case B) or punishment for attempted double spend (case C);
* **F1i** = fee offered to mine ADDR(i-1) // TO BE DELIBERATED: it could be better to just hardcode set F1 to always be equal F0, therefore economizing space for it in the blockchain;
* **F2i** = fee offered to mine BC in case of default;
* **T0i** = Block height of mining of BT;
* **T1i** = Block height owner should aim at broadcasting ADDR(i-1), or expected derivation of Ki from S1i. T1i = T0i+1 to T0i+6 blocks. This is to protect owner from dissensus. Namely, revealing ADDR(i-1) in a block and have it utilized to forge transaction in a competing block of same height.
* **T2i** = Block height of expiration of commitment. T2i = T0i+24*6 = T0i+144 (that is 24 h worth of blocks). This is to protect user from execution of commitment due to innocent network unavailability, at the same time put an incentive on miners to solve time-lock-puzzle (deriving Ki from S1i) to have F1i payable now rather than F2i later;

## Protocol

1. Alice broadcasts BTi;
2. Miner M0 includes BLi (but not BCi) in block of height T0i. Origin UTXO is frozen. TXi, contained in BLi, promises M0i F0i in fees, which is, however, not paid until steps 3 or 4 happen. M0 accepts to sell space in T0i for delayed payment because of commitment contained in BCi, which was not mined to save space on the blockchain;
3. A few blocks later, Alice broadcasts ADDR(i-1), which miner M1i includes in block of height T1i. That authenticates TXi by allowing the verification of LSIGi, conveied in BLi, mined but pending confirmation since T0i. Upon that happening, M0i finally gets F0i. Even if she doesn't, for whatever reason (possibly an adversary blocking her access to internet), miners can derive Ki from S1i, conveyed in BHi
M1i, miner of T1i gets their prescribed F1i. Origin UTXO is unfrozen;
4. In case T2i is reached without broadcasting of LAMPPRI, COMMITMENT becomes executable, and is, therefore mined. M2i, its miner gets F2i fees, and
M0i gets F0i+FF0i (to compensate for mining of BL and for the delay in payment). UTXOi is unfrozen.

## Security Analysis

There are three possible cryptanalysis to ADDR(i-1) in my scheme:

1.  Crack it from ADDRi alone before T0i (to be precise, before publishing of BTi);
2.  From ADDRi and (TXi, LVBSIGi) after T0i-1 (to be precise, after publishing of BTi);
3.  Outmine the rest of mining community starting from a disadvantage of not less than (T1-T0-1) blocks after T1 (to be precise, at time of publishing of ADDR(i-1));

For 1: a pre-image problem for a function
f1: {a | a is in the format of a ADDR} -> {a | a is in the format of a ADDR}, 
having both domain and codomain be the set of bitarrays in the same format as ADDRs. Here, f1 is given by:
f1(a) = h(PUB_FROM_PRI(KDF(a))), where PUB_FROM_PRI(.) is the derivation of public correspondent to private one.

For 2: a pre-image problem for a function
f2_(TXi,SALTi): {a | a is in the format of a ADDR} -> {a | a is in the format of a ADDR} x {s | s is in the format of a LVBSIG}
having as domain the set of bitarrays in the same format as ADDRs and codomain, the Cartesian product of the set of bitarrays in the same format as ADDRs with the set of bit arrays with the same length as LSIG. Here, f2_(TX,ECCPUB,SALT) is given by:
f2_(TXi,SALTi)(a) = (h(PUB_FROM_PRI(KDF(a))),h(TXi,ADDR(i-1),SALTi))

It can be argued that problem 2 is actually harder because attacker also has to mach the already published PUBi. But even the current statement of the problem is already unfeasible, particularly considering the limitted window of time (of T2-T0 block at best) for solving it.

For 3: Equivalente of a double-spending attack with, in the worst case, not less than (T1-T0-1) blocks in advantage for the rest of the community.

## Multi Use

Protocol clearly can be set to allow N usages, where N gives the total iterations of the Lamport chain. Through address reuse is generally discouraged for anonymity purposes, often time the opposite of anonymity, namely, the public biding between a name and a key, is desired. For cases like that, in particular, an additional economy is possible because of the coincidence of ADDR(i-1) of both Lamport pre-image and exchange address.
