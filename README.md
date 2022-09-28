# ZK Waitlist

The circuits and contracts in this repo describe the backend for a privacy-preserving waitlist based on zero knowledge proofs. A demo of this waitlist can be found at zkwaitlist.app, where you can deploy and test your own waitlist. 

We model a smart contract waitlist as a construct where users first join the waitlist (i.e. "claim" a spot), and then subsequently "redeem" their claimed spot. Because all Ethereum transactions are public, a typical smart contract waitlist suffers the issue that observers can see which accounts are claiming and/or redeeming certain spots on the waitlist. Most pertinently, an adversary can track the address that claimed a spot, see which address redeems that spot, and plausibly link the two addresses to the same party. The purpose of a privacy-preserving waitlist is to make it infeasible for onlookers to correlate the actions of claiming and redeeming to a shared address pool. 

zk-SNARKs enable privacy-preserving waitlist through a commitment-nullifier protocol. You can read more about this construction in Zcash's explainer on how SNARKs are used for privacy-preserving payments (https://z.cash/technology/zksnarks/). Essentially, users that join the waitlist are required to provide a commitment, which is a hashed version of a secret key. Redeeming a waitlist spot requires a user to provide a nullifier, which is a different hash of the same secret key. The idea is that you can only compute a commitment and nullifier corresponding to the same secret key if you actually know the secret key. How does the smart contract waitlist know that the commitment and nullifier provided by a user corresponds to the same secret? Well, it doesn't. It requires the user to also provide a zero knowledge proof that the commitment and nullifier were generated from the same secret. 

To elide the details, a zero knowledge proof enables a provider (in our case, the user) to demonstrably show the verifier (the waitlist smart contract) that it has performed a computation correctly. Furthermore, it can do so while keeping the inputs to the computation private (hidden from the verifier). For us, this means the user can show the smart contract waitlist that it does indeed know a secret key which maps to a commitment and nullifier, all without revealing the secret. The smart contract can be confident that this is the case as long as it verifies the zero knowledge proof provided by the user. 

Crucially, the privacy of the waitlist depends on the fact that onlookers cannot map certain commitments to certain nullifiers. Observers can see when a "claim" transaction or a "redeem" transaction is sent, but they cannot tell which "redeem" corresponds to which "claim". The zero knowledge proofs enable this privacy requirement to hold, since by design they require the user to only prove that the commitment corresponding to their secret exists in the Merkle tree of commitments represented in the contract. That is, the user provides the secret as well as the Merkle proof of inclusion to the zero knowledge proof generator, and the only result of the proof is to show that the secret maps to a valid node in the Merkle tree. 

The nullifiers play a critical role in the smart contract waitlist in that they prevent double-claiming. Once a nullifier is used to redeem a waitlist spot, it can never be "spent" again. Since each secret corresponds to a unique nullifier, that means a user that has claimed one spot on the waitlist cannot redeem it multiple times. 

## Circuits
There are two circom circuits which we use to describe the functions we generate zero knowledge proofs for. Each of these circuits comes with a corresponding smart contract which validates any proof generated from the circuit. 

**Locker:** After all users have joined the waitlist, any person can "lock" the waitlist. This ensures that nobody else can join the waitlist, and enables redemptions. The other purpose of the locker functionality is to learn the Merkle root of the commitment tree. In order to lock the waitlist, a user must provide a Merkle root corresponding to the Merkle tree of commitments stored in the waitlist, as well as a zero knowledge proof demonstrating that they computed the root correctly. The Locker circuit constructs a Merkle tree from its inputs and returns the Merkle root. Its inputs are public so that the smart contract waitlist can verify the inputs were indeed the list of commitments. 

**Redeemer:** The redeemer contract allows a user to prove that the have provided a valid nullifier for redemption. It takes as input a secret used to generate the commitment as well as a Merkle proof of inclusion in the commitment tree. First, the contract generates the nullifier as hash(secret, 1) and returns it as a public output. We use the SNARK-friendly Poseidon hash function in our implementation. Next, it computes the commitment as hash(secret, 0) and combines this with the Merkle proof to generate a Merkle root. The smart contract waitlist will verify the claimed Merkle root does indeed match its current Merkle root. Furthermore, if the nullifier has not been used and the proof is valid, it approves the redemption. 
