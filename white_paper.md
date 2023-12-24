# LVB Lamport-Scheme-Based Signature for Bitcoin Layer 1

We propose a hybrid-key signature scheme consisting of a conventional signing scheme with irrepudiable signature and [Lamport scheme](https://en.wikipedia.org/wiki/S/KEY). The benefit of it is a significant reduction of onchain footprint as will be demonstrated below.

## Notation

* **HL(.)** = serial-work- and memory-*hard* hash with *short* digest (ex.: Argon2 with ~ 12 bytes output. "L" for "Lamport");
* **HLL(.)** = serial-work- and memory-*hard* hash with *extremely long* digest (ex.: Argon2 with ~ 128MB output. "LL" for "Lamport" and "Long");
* **HA(.)** = nonspecific representation of conventional hashes with *long* (brute-force-resistant) digest length, and output formated into some Bitcoin address. "A" for "Address";
* **KDF(.)** = conventional key deriving function
* **SEED** = conventional Bitcoin seed
* **TAG** = conventional Bitcoin seed tag for deterministically deriving various keys from one seed.
* **ECCPUB** = public key correspondent to ECCPRI
* **ECCPRI** = KDF(SEED,TAG) //conventional BTC signing key (could be Schnorr)
* **LAMPPUB** = HLL(LAMPPRI,ECCPUB)
* **LAMPPRI** = HL(SEED,ECCPUB,TAG) //Though it is (more) feasible to crack a seed S that works as pre-image to LAMPRI, such seed can only be deemed valid if the public key correspondent to KDF(s) = ECCPUB, so ultimately, cracking seed is still as hard as cracking a conventional seed.
* **ADDR** = HA(ECCPUB,LAMPPUB) //Conventional BTC key fingerprint
* **TX** = plaintext transaction
* **SALT** = Contacenation of nonces of a few blocks recent by the time of issuing of TX, say T0-6, ... T-13
* **LSIG** = HL(TX,LAMPPRI,SALT)
* **COMMITMENT** = Smart contract stating "This UTXO is frozen until one of the following happens:
A) publishing of an L such that HL(TX,L,SALT) = LSIG before T2 in which case TX is deemed valid and executed, or B) T2 blocks from now, when miner of BL has gets additional FF0, and the miner M2 of COMMITMENT gets F2, both from UTXO"
* **BL** = "Bundle of Lamport scheme" = (TX, LSIG)
* **BC** = "Bundle of Commitment and Conventional Signing" = (COMMITMENT, ECCPRI(COMMITMENT), ECCPUB, LAMPPUB)	//LAMPPUB is added here to allow verification that ECCPUB corresponds to ADDR
* **BI** = "Initial Bundle" = (BL, BC)
* **BF** = "Final Bundle" = (LSIG,ECCPUB)
* **F0** = fee offered to mine BL
* **FF0** = fine offered to miner of BL to compensate for delay
* **F1** = fee offered to mine LAMPPRI // TO BE DELIBERATED: it could be better to just hardcode set F1 to always be equal F0, therefore economizing space for it in the blockchain.
* **F2** = fee offered to mine BC in case of default
* **T0** = Block height of broadcasting of BT
* **T1** = Block height owner should aim at broadcasting LAMPPRI. T1 = T0+1 to T0+6 blocks. This is to protect owner from dissensus (revealing LAMPPRI in a block and have it utilized to forge transaction in a competing block of same height).
* **T2** = Block height of expiration of commitment. T2 = T0+24*6 = T0+144 (that is 24 h worth of blocks). This is to protect user from execution of commitment due to innocent network unavailability.

## Protocol

1. Alice broadcasts BI;
2. Miner M0 includes BL (but not BC) in block of height T0. Origin UTXO is frozen. TX, contained in BL, promises M0 F0 in fees, which is, however, not paid until
steps 3 or 4 happen. M0 accepts to sell space in T0 for delayed payment because of commitment contained in BC, which was not mined
to save space on the blockchain;
3. A few blocks later, Alice broadcasts BF = (LAMPPRI, ECCPUB), which miner M1 includes in block of height T1. That
authenticates TX by allowing the verification of LSIG, conveied in BL, mined but pending confirmation since T0. Upon that happening, M0 finally gets F0.
M1, miner of T1 gets their prescribed F1. Origin UTXO is unfrozen;
4. In case T2 is reached without broadcasting of LAMPPRI, COMMITMENT becomes executable, and is, therefore mined. M2, its miner gets F2 fees, and
M0 gets F0+FF0 (to compensate for mining of BL and for the delay in payment). UTXO is unfrozen.

## Security Analysis

There are three possible cryptanalysis to LAMPPRI in my scheme:

1.  Crack it from ADDR alone before T0 (to be precise, before publishing of BI);
2.  From ADDR and (TX, SIG) after T0-1 (to be precise, after publishing of BI);
3.  Outmine the rest of mining community starting from a disadvantage of not less than (T1-T0-1) blocks after T1 (to be precise, at time of publishing of LAMPRI);

For this analysis, we take note of the ironic fact that *LAMPPUB is never mined*. For that reason we can have LAMPPUB be 1Mb or even 1Gb long aiming 
at having rate of collision in HL(.) be negligible (note this is perfectly adherent to the proposition of memory-hard-hash, and would have the additional 
benefit of allowing processing-storage trade-off). In this case, we really have:

For 1: a pre-image problem for a function
f1: {k| k is a valid ECCPRI} X {l | l is a valid LAMPPRI} -> {a | a is in the format of a ADDR}, 
having as domain the Cartesian product of the set of possible ECCPRIs with the set of possible LAMPPRIs and codomain, 
the set of bitarrays in the same format as ADDRs. Here, f1 is given by:
f1(k,l) = HC(HLL(pub_from_pri(k),HL(l))), where pub_from_pri(.) is the derivation of public correspondent to private one.


For 2: a pre-image problem for a function
f2_(TX,ECCPUB,SALT): {l | l is 'a valid LAMPPRI'} -> {a | a is 'in the format of ADDRs'} X {s | s is in the format of a LSIG}
having as domain the set of possible LAMPPRIs and codomain, the Cartesian product of the set of bit arrays in the same format as ADDRs with the set of 
bit arrays with the same length as LSIG. Here, f2_(TX,ECCPUB,SALT) is given by:
f2_(TX,ECCPUB,SALT)(l) = (HC(HLL(ECCPUB,l)),HL(SALT)

Notice that, whatever advantage of attack 2 might have over 1 due to smaller domain has to overcome the disavantage of having only approximately (T1-T0) 
blocks worth of time to be carried out.

For 3: Equivalente of a double-spending attack with, in the worst case, not less than (T1-T0-1) blocks in advantage for the rest of the community.

## Multi Use

So far, the protocol allows for only one instance of LVB signature. ADDR could still receive transactions, but in order to spend them ulterior transactions 
would have to be signed with ECCPRI, instead, with conventional footprint. This, in one hand incentivizes the anonymity practice of non-key reuse. That benefit 
could be enhanced by protocol forbiding incoming transactions to ADDR after publishing of LAMPPRI.

There is however, another possible approach, that contemplates the use case for which the opposite of anonymity is desired, and which further reduces the 
on-chain footprint: a hybrid hash / key derivation chain with LAMPPRI_i, the ith LAMPPRI actually consisting of ADDR_(i-1), the (i-1)th ADDR.

TO BE CONTINUED
