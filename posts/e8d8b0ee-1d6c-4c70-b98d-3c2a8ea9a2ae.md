# Bulletproofs



bullet proofs don't hide who is paying who. It would allow you to encrypted the transaction so that you can't see the amounts?


What do you need to verify for a transaction to be valid.

Still use Pedersen commitments which are discrete log based.

## Validity of a Bitcoin Transaction

1. Signature is Correct 
2. Inputs are unspent 
3. The sum of the inputs are equal to the sum of the outputs plus the fees

On the third point you care that both the inputs are positive. Otherwise the sum adding up is meaningless. Range Proof.

Non-Interactive Zero Knowledge Proof of Knowledge (NIZK). At any point in the future someone can verify the proof is valid.


I believe that Pedersen commitment is the name of the encryption scheme for confidential transactions, for which bullet proofs are just a part of the verification.

Types of range proofs: SNARKs (Trusted Setup), CT Range proof (Large proof size), and Bullet proof (Log size, only discrete log assumption).

Bullet proof still linear time verification. 

Constant time implementations to prevent timing attacks.


In 2017, Bünz, Bootle, Boneh, Poelstra, Wuille, and Maxwell published Bulletproofs, which dramatically improves proof performance both in terms of proof size and verification time. In addition, it allows for proving a much wider class of statements than just range proofs.

Pedersen commitment, which has the additional property of being additively homomorphic, which means that Com(a) + Com(b) = Com(c) only if a + b = c.\




## Trusted Setups 
bad

Verifiable Shuffle, without trusted setup

https://www.youtube.com/watch?v=sgruTaH_w1s


This would break all bitcoin mining ASICs, as they would not be able to verify the transactions? 
