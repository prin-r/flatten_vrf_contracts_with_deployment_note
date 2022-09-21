#  Band verifiable random function (VRF)

The foundation of the randomness matters for security reasons, many modern distributed applications (dApps) often rely on "good" randomness, generated freshly and independently of the application's state. In addition, the random result should have integrity, which means it should be verifiable, can't be tampered with, and is unpredictable. For example, consider an online lottery where participants place their bids, a random result, and winnings are returned according to the bid placement. Likewise, leader elections (often used in committee-based blockchain platforms) proceed in rounds to randomly elect a leader among participants. In such applications, it is typically crucial to guarantee that randomness is sampled uniformly and independently of the application's state (which makes it "hard to predict"), which will ensure that there are no malicious actors can affect the leader election or no lottery player can predict the result better than guessing at random.

### Verifiable Random Function (VRF)

Applications that are required to produce “good” random values commonly rely on cryptography techniques to deliver pseudorandom values, i.e., impossible to distinguish from uniformly random ones for all practical purposes. A verifiable random function (VRF) is a mathematical operation that takes some input and produces a pseudorandom output along with the proof of authenticity of the output generation process. Challengers can verify the proof to ensure the pseudorandom result is valid and does not come out of thin air.

In general, the core of the VRF system will have a set of secret keys used to generate verifiable results and a set of corresponding public keys used to verify the results produced. For example, given a secret key generated in private and a cryptographic function that maps a seed to an output value with its proof. The crucial property is that someone that does not have access to the secret key cannot distinguish in polynomial time the output from a value that is sampled uniformly at random from the range of all possible outcomes.

##### VRF security properties

- Unpredictability: This ensures that the computed outputs are distributed in a way that is, for all practical purposes, uniformly random. It is a fundamental property of a VRF, as it says that the VRF behaves like a random oracle. In practice, this implies that anyone not knowing the secret key has no way to predict the outcome that is better than “randomly guessing” even when knowing the seed. So, if the input seeds are chosen with sufficiently high entropy, it is practically impossible to predict the output.
- Uniqueness: This ensures that, after the VRF providers publish their secret key, they can only produce proof that will convince others of the correct VRF output value for every seed. In other words, for a given (secret key, seed), it is incredibly hard to find two different VRF values, both of which pass the verification. This property is crucial to protect against a cheating actor that tries to claim a specific output other than the correct one for their purposes.
- Collision-Resistance: This ensures that it is computationally hard to find two different inputs, "seed1" and "seed2", with the same secret key to obtain the identical output value —much like the classic property of cryptographic hash functions. The difference is that for VRFs, this holds even against an adversary that knows the secret key. Note that this offers a different type of protection than the unique property. For example, it protects against a party that tries to claim an output computed from one input seed as if it was computed from a different "seed2".

### BandChain VRF

Our solution for verifiably (pseudo-)randomness is based on the BandChain blockchain. Our protocol uses a verifiable random function (VRF) to cryptographically secure that produced results have not been tampered with. We will present in detail how our protocol operates in this document.

Bandchain Verifiable Randomness extends the general form of the VRF system to serve randomness requests for dApps, which is based on the distributed Bandchain oracle network. Bandchain is a public blockchain that provides APIs for data and services stored “off-chain” on the traditional web or from third-party providers. It supports generic data requests from other public blockchains and performs on-chain aggregation via Bandchain oracle scripts. The aggregation process works like a smart contract on the EVM platform, executed on-chain to produce the oracle results. The oracle results are also stored on the chain. After that, the results will be returned to the calling dApp on the main blockchain (typically Ethereum or other EVMs), accompanied by a proof of authenticity via customized one-way bridges (or via an Inter-Blockchain Communication protocol). To guarantee verifiably “good” randomness suitable for security-critical applications, we deploy the cryptographic primitive of verifiable random functions. At a high level, a VRF provides values that are indistinguishable from uniformly random ones and can be verified for their authenticity concerning a pre-published public key.

We chose the VRF of this [paper](https://eprint.iacr.org/2017/099), which has already been adopted in various other protocols. The construction is based on a well-studied cryptographic hardness assumption over prime-order elliptic curve groups. For our instantiation, we chose the widely-used Ed25519 curve that achieves very good performance and has a transparent choice of parameters, as well as the Elligator, for our hash-to-curve installation. Our implementation is fully compliant with the [VRF draft standard](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-vrf-09). Moreover, we implemented all the necessary techniques to achieve full security. For completeness, we include the pseudo-code description of our VRF below.

![figure1](https://user-images.githubusercontent.com/12705423/189547735-895f4cd1-ad5b-4c29-b84a-1cb6fa5f0c23.png)

##### Protocol Flow

![figure2](https://user-images.githubusercontent.com/12705423/161716790-8696406a-af8d-422b-8ff4-5092cae4d0e1.png)

At a high level, our protocol works as follows. First, two contracts are deployed on the main chain (Ethereum or other EVMs), VRF contract and Bridge. The first is in charge of receiving randomness requests from dApps and contains code that pre-processes the request to be ready for submission to the Band side-chain. It also works as the receiving end of the request’s result. The second, as the name denotes, works as the connecting “bridge” between the two chains to validate the latest state of the side chain and verify that the received results for VRF requests are indeed the ones computed and stored on BandChain. 
A third-party dApp that wishes to request a random value submits its request to the VRF contract, which will then prepare the actual VRF input by expanding it into a VRF seed. This is then picked up by incentivized actors and/or the Band foundation and is submitted as a VRF request to BandChain. In particular, a VRF oracle Script collects this request and then maps it to the set of VRF data sources available to the chain, as well as a number of BandChain validators. Then the oracle script randomly assigns it to a VRF provider corresponding to one of the VRF data sources. The assigned provider evaluates the VRF on the prescribed input using its VRF secret key and then broadcasts the result to the Band network. Next, all chosen validators run the VRF verification algorithm using this provider’s public key and, if verification succeeds, transmit the result to the VRF oracle script. Finally, after collecting the necessary number of results from the validators, the oracle script accepts the majority as the final result, which becomes part of the BandChain state. After that, the final result will get included in the next block’s computation.
The final result is transmitted back to the main chain’s VRF contract with a Merkle tree proof for its inclusion on the BandChain’s state. This proof is then verified with the Bridge contract. After successful checking, the final result is finally returned to the original dApp.






