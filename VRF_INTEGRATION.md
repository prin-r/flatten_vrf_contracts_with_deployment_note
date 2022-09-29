# VRF integration

In this section, we will discuss how to integrate your smart contracts with the Band VRF. This section is separated into two sub-sections: **Requesting** and **Resolving**.

Typically, when building on-chain applications that rely on an unpredictable outcome, such as lottery apps or games, the system requires a reliable source of randomness, and that is when the Band VRF comes into play.

---


### Requesting

![img1](https://user-images.githubusercontent.com/12705423/192215486-fcf23603-19df-4c04-ab2f-2fa56fc05c53.jpg)

As shown in the VRF workflow, a request and callback model is used to obtain new random data. Two transactions are required in order to complete the process. The first transaction is the transaction that contains a consumer's request for random data, and the second transaction is the transaction that resolves the request.

Let us assume that you are building an on-chain application that uses the Band VRF as the reliable source of randomness. Your contract(s) must reference the VRF Provider contract in order to request random data.

First, you need an interface for the VRFProvider contract.
```solidity=
interface IVRFProvider {
    /// @dev The function for consumers who want random data.
    /// Consumers can simply make requests to get random data back later.
    /// @param seed Any string that used to initialize the randomizer.
    function requestRandomData(string calldata seed) external payable;
}
```

Then, the consumer only needs to call `requestRandomData` with a string parameter called `seed`.

**For security reasons, the seed is a generated string on the consumer side.
Each consumer can use each seed only once because the VRFProvider contract implements a mapping to track used seeds for each consumer address.**

```solidity=
// Mapping that enforces the client to provide a unique seed for each request
mapping(address => mapping(string => bool)) public hasClientSeed;
```

For example, let us assume that there are two consumer contracts: *Contract A* and *Contract B*. *Contract A* requests the VRFProvider contract with the seed `AAA`, and *Contract B* requests the VRFProvider contract with the seed `BBB`. After both requests have been successfully made, *Contract A* can no longer use `AAA` as the seed again. Similarly, *Contract B* can no longer use `BBB` as the seed again. However, *Contract A* can still use `BBB` as the seed, and *Contract B* can still use `AAA` as the seed.

After including the `IVRFProvider`, the consumer can now make a request-call to the VRFProvider contract, as shown in the example implementation below.

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

When calling `requestRandomData(seed)`, the consumer can specify `msg.value` to incentivize others to resolve the random data request. However, the consumer can choose not to provide any incentive and resolve the request themselves.

Apart from implementing the request function to the VRFProvider contract, the consumer is required to implement a callback function for the VRFProvider contract to call back and do something with the random result.

The example below is an implementation that shows how to implement the callback function `consume`. 

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

As shown above, the `consume` function implements a logic that verifies whether the caller is the VRFProvider contract or not. This is to ensure that no one can call this function except the VRFProvider contract. With regards to the remaining logic in the example, the callback function only saves the callback data from the VRFProvider contract to its state.

Our deployed `Bridge`, `VRFProvider`, `MockVRFConsumer` and `VRFLens`contracts on each chain are shown in the table below.

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


---


### Resolving

Anyone can resolve any unresolved requests in the VRFProvider contract by tracking the mapping called `tasks` in the VRFProvider contract.

The code below is a simplified version of the VRFProvider contract that shows what `tasks` looks like.

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

When the resolver (this can be a self-implemented worker bot or a bounty hunter) finds an unresolved request, the resolver can resolve it by making a request transaction on the BandChain for the VRF randomness. After the random result is finalized on the BandChain, the resolver can retrieve the proof of inclusion of the result, and then relay the proof via a `relayProof` function on the VRFProvider contract. The resolver also needs to specify the nonce of the task that it wants to resolve.

#### Manually request and resolve

This section will demonstrate how to request random data from the VRF provider and then resolve the request manually using [Goerli](https://goerli.etherscan.io/) and [Laozi-Testnet5](https://laozi-testnet5.cosmoscan.io/) UI.

Firstly, go to the [VRF provider contract on Goerli](https://goerli.etherscan.io/address/0xF1F3554b6f46D8f172c89836FBeD1ea8551eabad#readContract) to view some of its configuration.

![img2](https://user-images.githubusercontent.com/12705423/193009074-542be3ea-8b6f-4f12-b96f-1b153afdd2a8.png)

The image above shows three parameters for requesting data on Bandchain: `oskCount`, `minCount`, and `oracleScriptID`.

Let's move to the `Write Contract` tab to begin requesting data on the Ethereum side.

[![video1](https://i9.ytimg.com/vi/3ki0LhArdoI/mq2.jpg?sqp=CISJ15kG&rs=AOn4CLDxoB24651sejcUvK84IQry9TsO5g)](https://youtu.be/3ki0LhArdoI)

The video above shows that anyone can request Band VRF randomness by calling `requestRandomData` on the VRF provider contract. After the calling is successful, the VRF provider will log some parameters that will be used to make a request on the Bandchain side.

<i>See the request transaction [here](https://goerli.etherscan.io/tx/0x640425325d7fa5f7b56a9d966f75863a40e8a1139ddf5a77b55a66dd8d03ba46#eventlog)</i>

![img_task_1](https://user-images.githubusercontent.com/12705423/193093565-bc89e14b-b357-4dda-88f8-788b56c5eff0.png)

The image above shows the current stage of the task/request that we just **created**.

The next step is making a request on Bandchain using those parameters from the previous steps. 

[![video1](https://i9.ytimg.com/vi/oxbLmLv_fgQ/mq1.jpg?sqp=COCU15kG&rs=AOn4CLAHohDR_wa6ZcaOZ8BhgevUox85dw)](https://youtu.be/oxbLmLv_fgQ)

In this case, we will go to [oracleScriptID 152 or O152](https://laozi-testnet5.cosmoscan.io/oracle-script/152#execute) and then fill in the `seed`, `time`, `workerAddress`, `minCount`, and `askCount`, as shown in the video above.

<i>The worker address can be any EOA address that the resolver will use for relaying to resolve the request on the Ethereum side.</i>

The next step is to copy the Merkle proof and then switch back to the Ethereum side to relay the proof on the VRF provider contract.

[![video1](https://i9.ytimg.com/vi/uFDPKyerWLY/mq3.jpg?sqp=COSb15kG&rs=AOn4CLA6_6aLgcAk6et6-XR2-rIDk48rAg)](https://youtu.be/uFDPKyerWLY)

<i>See the relay transaction [here](https://goerli.etherscan.io/tx/0x5e7d85d4d2cd41b71f3ea35fff2a17dba8d19c085678d6f8dd7e22ca664da9b8#eventlog)</i>

![img_task_2](https://user-images.githubusercontent.com/12705423/193094318-67cee15a-220d-4122-b7bb-79f64158a0c9.png)

The image above shows the current stage of the task/request that we just **resolved**.
