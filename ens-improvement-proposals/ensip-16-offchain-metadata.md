---
description: Allows metadata to be queried on EIP-3668 enabled names
---

# ENSIP-16: Offchain metadata

| **Author**    | Jeff Lau \<jeff@ens.domains>, Makoto Inoue \<makoto@ens.domains> |
| ------------- | ---------------------------------------------------------------- |
| **Status**    | Draft                                                            |
| **Submitted** | 2022-09-22                                                       |

### Abstract

This ENSIP specifies APIs for querying metadata directly on the resolver for EIP-3668 (CCIP Read: Secure offchain data retrieval) enabled names. EIP-3668 will power many of the domains in the future, however since the retrieval mechanism uses wildcard + offchain resolver, there is no standardised way to retrieve important metadata information such as the owner (who can change the records), or which L2/offchain database the records are stored on.

### Motivation

With EIP-3668 subdomains already starting to see wide adoption, it is important that there is a way for frontend interfaces to get important metadata to allow a smooth user experience. For instance a UI needs to be able to check if the currently connected user has the right to update an EIP-3668 name.

This ENSIP addresses this by adding a way of important metadata to be gathered on the offchain resolver, which would likely revert and be also resolved offchain, however there is an option for it to be also left onchain if there value was static and wouldn't need to be changed often.

### Specification

The metadata should include 2 different types of info

- Offchain data storage location related info: `graphqlUrl` includes the URL to fetch the metadata.

- Ownership related info: `owner`, `isApprovedForAll` defines who can own or update the given record.

#### Context

An optional field "context" is introduced by utilizing an arbitrary bytes string to define the namespace to which a record belongs.

For example, this "context" can refer to the address of the entity that has set a particular record. By associating records with specific addresses, users can confidently manage their records in a trustless manner on Layer 2 without direct access to the ENS Registry contract on the Ethereum mainnet. Please refer to [ENS-Bedrock-Resolver](https://github.com/corpus-io/ENS-Bedrock-Resolver#context) for the reference integration

#### Dynamic Metadata

Metadata serves a crucial role in providing valuable insights about a node owner and their specific resolver. In certain scenarios, resolvers may choose to adopt diverse approaches to resolve data based on the node. An example of this would be handling subdomains of a particular node differently. For instance, we could resolve "optimism.foo.eth" using a contract on optimism and "gnosis.foo.eth" using a contract on gnosis.
By passing the name through metadata, we empower the resolution process, enabling CcipResolve flows to become remarkably flexible and scalable. This level of adaptability ensures that our system can accommodate a wide array of use cases, making it more user-friendly and accommodating for a diverse range of scenarios.

### Implementation

#### L1

```solidity

// To be included in
// https://github.com/ensdomains/ens-contracts/blob/staging/contracts/resolvers/Resolver.sol
interface IOffChainResolver {
    // optional.
    // this returns data via l2 with EIP-3668 so that non EVM chains can also return information of which address can update the record
    // The same function name exists on L2 where delegate returns address instead of bytes
    function isApprovedFor(bytes context, bytes32 node, bytes delegate) returns (bool);

    /** @dev Returns the metadata about the node
     * @return name can be l2 chain name or url if offchain
     * @return coinType according to https://github.com/ensdomains/address-encoder
     * @return graphqlUrl url of graphql endpoint that provides additional information about the offchain name and its subdomains
     * @return storageType 0 = EVM, 1 = Non blockchain, 2 = Starknet
     * @storageLocation = When storageType is 0, l2 contract address. When storageType is 1, url endpoint in bytes format
     * @return context = an arbitrary bytes string to define the namespace to which a record belongs such as the name owner.
     */
    function metadata(bytes calldata name)
        external
        view
        returns (string memory, uint256, string memory, uint8, bytes memory, bytes memory)
    {
        return (name, coinType, graphqlUrl, storageType, storageLocation, context);
    }

    // Optional. If context is dynamic, the event won't be emitted.
    event MetadataChanged(
        string name,
        uint256 coinType,
        string graphqlUrl,
        uint8 storageType,
        bytes storageLocation,
        bytes context
    );
}
```

#### L2 (EVM compatible chain only)

```solidity
// To be included in the contract returned by `metadata` function `storageLocation`
interface IL2Resolver {
    /**
     * @dev Check to see if the delegate has been approved by the context for the node.
     *
     * @param context = an arbitrary bytes string to define the namespace to which a record belongs such as the name owner.
     * @param node
     * @param delegate = an address that is allowed to update record under context
     */
    function isApprovedFor(bytes context,bytes32 node,address delegate) returns (bool);

    event Approved(
        bytes context,
        bytes32 indexed node,
        address indexed delegate,
        bool indexed approved
    );
}
```

```javascript
const node = namehash('ccipreadsub.example.eth')
const resolver = await ens.resolver(node)
const owner = await resolver.owner(node)
// 0x123...
const dataLocation = await.resolver.graphqlUrl()
// {
//   url: 'http://example.com/ens/graphql',
// }
```

##### GraphQL schema

[GraphQL](https://graphql.org) is a query language for APIs and a runtime for fulfilling those queries with onchain event data. You can use the hosted/decentralised indexing service such as [The Graph](https://thegraph.com), [Goldsky](https://docs.goldsky.com/introduction), [QuickNode](https://marketplace.quicknode.com/add-on/subgraph-hosting) or host your own using The Graph, or [ponder](https://ponder.sh)

##### L1

`Metada` is an optional schema that indexes `MetadataChanged` event.

```graphql

type Domain @entity{
  id
  metadata: Metadata
}

type Metadata @entity {
  "l1 resolver address"
  id: ID!
  "Name of the Chain"
  name: String
  "coin type"
  coinType: BigInt
  "url of the graphql endpoint"
  graphqlUrl: String
  "0 for evm, 1 for non blockchain, 2 for starknet"
  storageType: Int
  "l2 contract address"
  storageLocation: Bytes
  "optional field to store an arbitrary bytes string to define the namespace to which a record belongs"
  context: Bytes
  "optional field if the name has expirty date offchain"
  expiryDate: BigInt
}

```

##### L2

L2 graphql URL is discoverable via `metadata` function `graphqlUrl` field.
Because the canonical ownership of the name exists on L1, some L2/offchain storage may choose to allow multiple entities to update the same node namespaced by `context`. When quering the domain data, the query should be filtered by `context` that is returned by `metadata`function `context` field

```graphql
type Domain {
  id: ID! # concatenation of context and namehash delimited by `-`
  context: Bytes
  name: String
  namehash: Bytes
  labelName: String
  labelhash: Bytes
  resolvedAddress: Bytes
  parent: Domain
  subdomains: [Domain]
  subdomainCount: Int!
  resolver: Resolver!
  expiryDate: BigInt
}

type Resolver @entity {
  id: ID! # concatenation of node, resolver address and context delimited by `-`
  node: Bytes
  context: Bytes
  address: Bytes
  domain: Domain
  addr: Bytes
  contentHash: Bytes
  texts: [String!]
  coinTypes: [BigInt!]
}
```

### Offchain storage service

[EIP-5559](https://eips.ethereum.org/EIPS/eip-5559#data-stored-in-an-off-chain-database) has already been proposed to handle both L2 and off chain storage by utilizing the similar method to [EIP-3668](https://eips.ethereum.org/EIPS/eip-3668).
While this is technically feasible, it adds extra function calls simply to discover how to update the data. The simpler alternative for ENS related storage update is to query storage service endpoint from `storageLocation` in bytes format when `storateType` is set to `1`. The client then make a HTTP POST request to the storage location with signature information.

The body attached to the request is a JSON object that includes sender, data, inception date and a signed copy of the abi encoded data, sender, and inception date.

Example HTTP POST request body including requestParams and signature:

```
  const name = 'vitalik.eth'
  const node = ethers.utils.namehash(name)
  const iface = new ethers.utils.Interface(['function setAddr(bytes,address)'])
  const signer = signers[0]
  const signerAddress = signer.address
  const data = iface.encodeFunctionData('setAddr',[node, signerAddress])
  const block = await ethers.provider.getBlock('latest')
  inceptionDate = block.timestamp
  signature = await signers[0].signMessage(
    ethers.utils.arrayify(
      keccak256(
        ['bytes', 'address', 'uint256'],
        [
          data,
          signer,
          inceptionDate
        ],
      ),
    ),
  )
  const request = {
    data:,
    sender:signer,
    inceptionDate,
    signature
  }
```

Once the storage service receives the request, it decodes the sender information from the signed signature and perform the data update if the `sender` matches the name onwer defined by the optional `context` field.

### Backwards Compatibility

None

### Open Items

- Should `owner` and `isApprovedForAll` be within graphql or shoud be own metadata function?

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).