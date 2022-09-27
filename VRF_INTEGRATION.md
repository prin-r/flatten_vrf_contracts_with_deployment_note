# VRF integration

This part will discuss integrating your smart contracts with Band's VRF. We will separate this section into two sub-sections: requesting and resolving.

Typically when building on-chain applications that rely on an unpredictable outcome, such as lottery apps or games, the system needs a reliable source of randomness, and that's when Band's VRF comes into play.

### Requesting

![img](https://user-images.githubusercontent.com/12705423/192215486-fcf23603-19df-4c04-ab2f-2fa56fc05c53.jpg)

According to the VRF workflow, the process for getting new random data is a request and callback model. In order to complete the process, there will be two transactions. The first is the transaction that contains a consumer's request for random data, and the second is the transaction that resolves the request.

Let's assume that you are building an on-chain application that uses the VRF as a reliable source of randomness. Your contract or contracts must have a reference to Band's VRF provider to be able to request the VRF provider when there is a need for random data.

First, you need an interface for the VRF provider.
```solidity=
interface IVRFProvider {
    /// @dev The function for consumers who want random data.
    /// Consumers can simply make requests to get random data back later.
    /// @param seed Any string that used to initialize the randomizer.
    function requestRandomData(string calldata seed) external payable;
}
```

Then, the consumer only needs to call "requestRandomData" with a string parameter we call seed.

**For security reasons, the seed is a generated string on the consumer side.
One consumer can only use one seed once as the VRF provider has a mapping to track used seeds for each consumer address.**

```solidity=
// Mapping that enforces the client to provide a unique seed for each request
mapping(address => mapping(string => bool)) public hasClientSeed;
```

For example, let's assume that there are two consumer contracts: A and B. Contract A request the provider with seed AAA, and contract B request the provider with seed BBB. After both requests were successfully made, contract A can't use AAA as seed again, and the same B can't use BBB again. However, A can still use BBB, and the same as B can still use AAA.

After including the IVRFProvider, the consumer should be able to make a request-call the provider contract as in the example implementation below.

```solidity=
contract MockVRFConsumer {
    IVRFProvider public provider;

    constructor(IVRFProvider _provider) {
        provider = _provider;
    }

    function requestRandomDataFromProvider(string calldata seed) external payable {
        provider.requestRandomData{value: msg.value}(seed);
    }
}
```

When calling "requestRandomData(seed)", the consumer can specify "msg.value" to incentivize others to resolve the random data request. However, consumers can choose not to provide any incentive and resolve the request themselves.

After the consumer can make a call to the provider, the consumer still needs to implement the callback function for the provider to be able to call back and do something with the random result.

The example below is an implementation that shows how to implement the callback function. 

```solidity=
contract MockVRFConsumer {
    IVRFProvider public provider;
    string public latestSeed;
    uint64 public latestTime;
    bytes32 public latestResult;

    constructor(IVRFProvider _provider) {
        provider = _provider;
    }

    function requestRandomDataFromProvider(string calldata seed) external payable {
        provider.requestRandomData{value: msg.value}(seed);
    }
    
    function consume(string calldata seed, uint64 time, bytes32 result) external override {
        require(msg.sender == address(provider), "Caller is not the provider");
        
        latestSeed = seed;
        latestTime = time;
        latestResult = result;
    }
}
```

As you can see, the consume function needs to have a logic that verifies if the caller is the provider to ensure that no one can call this function except the provider. For other logic in the example, the callback function only saves the callback data from the provider to its state.

You can see our deployed MockVRFConsumer on each chain in the table below.

#### Testnet

|Chain \ Contract |Bridge|VRFProvider|MockVRFConsumer|VRFLens|
|-----------------|------------------------------------------|------------------------------------------|------------------------------------------|------------------------------------------|
|[Goerli](https://goerli.etherscan.io)          |0x6f057CE91CFcB59d839Db91e8DF067278a704cb8|0xF1F3554b6f46D8f172c89836FBeD1ea8551eabad|0x6aFCBD05f4718B994a290cfF03547DDFFcd74E08|0x6e876b4Ed458af275Eb049a3f89BF0909618d154|
|[Rinkeby](https://rinkeby.etherscan.io)          |0xB8651240368f64aF317c331296b872b815892E00|0xfdBBAD9D6A4e85a38c12ca387014bd5F697f0661|0xf48F60A97b1BDf0D47fa460a0894634124d039b4|0xD0F7DcDaC3CCaB2f64b97CaEEa6ebDe79a6a93e2|
|[Cronos](https://testnet.cronoscan.com)           |0x6f057CE91CFcB59d839Db91e8DF067278a704cb8|0xE2f7Cf77DF70af8e92FF69B8Ffc92585C307a358|0x6aFCBD05f4718B994a290cfF03547DDFFcd74E08|0xdcFA1244c37262441AA7caF9893fdD99dB101E2A|
|[OKC](https://www.oklink.com/en/okc-test)              |0xF22bA22A57d387F3F55B4d7643092338cCDf99D5|0x6afcbd05f4718b994a290cff03547ddffcd74e08|0xbf59aA508bABFA3B112553E05b45dcdB21997891|0xB8651240368f64aF317c331296b872b815892E00|
|[Oasis](https://testnet.explorer.emerald.oasis.dev)            |0xee8346E77d73730e40Cac18cd8812a2f2e0496de|0x4ADE1059F424673B0d660cD87A733b940d309bcF|0x74865F64aCaF86cD8dfa0c185bE177085106C91a|0x7f38DF2403c0E767662B5ABB09e4c86A8FDD1869|
|[BSC](https://testnet.bscscan.com)              |0x4ADE1059F424673B0d660cD87A733b940d309bcF|0x74865F64aCaF86cD8dfa0c185bE177085106C91a|0x7f38DF2403c0E767662B5ABB09e4c86A8FDD1869|0x7c3D5a83a335CED7b6b6beaa959DaD416ae88f27|
|[OP Goerli](https://goerli-optimism.etherscan.io) |0x6f057CE91CFcB59d839Db91e8DF067278a704cb8|0xF1F3554b6f46D8f172c89836FBeD1ea8551eabad|0xE2f7Cf77DF70af8e92FF69B8Ffc92585C307a358|0x3ffBc08b878D489fec0c80fa65C9B3933B361764|
|[Polygon](https://mumbai.polygonscan.com) |0x2DE50E85D5F11DF9b33Da42e3174611CeB9461d9|0x0173cE38C64Be34e7f23f39346c2D9AF5d9743FB|0xFb4d5252ca8FAFaE3Fe8718a9eE8bcF72266589F|0x14919325f2d97a05d146b7b4c9374b265e722f00|
|[Avalanche](https://testnet.snowtrace.io)  |0x6f057CE91CFcB59d839Db91e8DF067278a704cb8|0xF1F3554b6f46D8f172c89836FBeD1ea8551eabad|0xE2f7Cf77DF70af8e92FF69B8Ffc92585C307a358|0x3ffBc08b878D489fec0c80fa65C9B3933B361764|

#### Mainnet

|Chain \ Contract |Bridge|VRFProvider|VRFComsumer|
|-----------------|------|-----------|-----------|
|                 |      |           |           |


### Resolving

Anyone can resolve every non-resolved request in the provider contract by tracking the mapping called "tasks" in the provider contract.

The code below is a simplified version of the VRFProvider that shows what "tasks" looks like.

```solidity=
contract VRFProvider {
    
    struct Task {
        bool isResolved;
        uint64 time;
        address caller;
        uint256 taskFee;
        bytes32 seed;
        string clientSeed;
        bytes proof;
        bytes result;
    }
    
    uint64 public taskNonce;

    // Mapping from nonce => task
    mapping(uint64 => Task) public tasks;

}
```

When the resolver(worker, bounty hunter, or whatever you want to call) finds a non-resolved request, they can resolve it by making a request transaction on Bandchain for VRF randomness. After the random result is finalized on Bandchain, the resolver can copy the proof of availability for the result and then relay it via a "relayProof" function on the VRFProvider. The resolver also needs to specify a nonce of the task they want to resolve.
