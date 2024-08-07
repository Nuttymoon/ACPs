| ACP | 20 |
| :--- | :--- |
| **Title** | Ed25519 p2p |
| **Author(s)** | Dhruba Basu ([@dhrubabasu](https://github.com/dhrubabasu)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/21))|
| **Track** | Standards |

## Abstract

Support Ed25519 TLS certificates for p2p communications on the Avalanche network. Permit usage of Ed25519 public keys for Avalanche Network Client (ANC) NodeIDs. Support Ed25519 signatures in the ProposerVM.

## Motivation

Avalanche Network Clients (ANCs) rely on TLS handshakes to facilitate p2p communications. AvalancheGo (and by extension, the Avalanche Network) only supports TLS certificates that use RSA or ECDSA as the signing algorithm and explicitly prohibits any other signing algorithms.

If a TLS certificate is not present, AvalancheGo will generate and persist to disk a 4096 bit RSA private key on start-up. This key is subsequently used to generate the TLS certificate which is also persisted to disk. Finally, the TLS certificate is hashed to generate a 20 byte NodeID. Authenticated p2p messaging was required when the network started and it was sufficient to simply use a hash of the TLS certificate. With the introduction of Snowman++, validators were then required to produce shareable message signatures. The Snowman++ block headers (specified [here](https://github.com/ava-labs/avalanchego/blob/v1.10.15/vms/proposervm/README.md#snowman-block-extension)) were then required to include the full TLS `Certificate` along with the `Signature`.

However, TLS certificates support Ed25519 as their signing algorithm. Ed25519 are IETF recommendations ([RFC8032](https://datatracker.ietf.org/doc/html/rfc8032)) with some very nice properties, a large one being their size:

- 32 byte public key
- 64 byte private key
- 64 byte signature

Because of the small size of the public key, it can be used for the NodeID directly with a marginal hit to size (an additional 12 bytes). Additionally, the brittle reliance on static TLS certificates can be removed. Using the Ed25519 private key, a TLS certificate can be generated in-memory on node startup and used for p2p communications. This reduces the maintenance burden on node operators as they will only need to backup the Ed25519 private key instead of the TLS certificate and the RSA private key.

Ed25519 has wide adoption, including in the crypto industry. A non-exhaustive list of things that use Ed25519 can be found [here](https://ianix.com/pub/ed25519-deployment.html). More information about the Ed25519 protocol itself can be found [here](https://ed25519.cr.yp.to).

## Specification

### Required Changes

1. Support registration of 32-byte NodeIDs on the P-chain
2. Generate an Ed25519 key by default (`staker.key`) on node startup
3. Use the Ed25519 key to generate a TLS certificate on node startup
4. Add support for Ed25519 keys + signatures to the proposervm
5. Remove the TLS certificate embedding in proposervm blocks when an Ed25519 NodeID is the proposer
6. Add support for Ed25519 in `PeerList` messages

Changes to the p2p layer will be minimal as TLS handshakes are used to do p2p communication. Ed25519 will need to be added as a supported algorithm.

The P-chain will also need to be modified to support registration of 32-byte NodeIDs. During serialization, the length of the NodeID is not serialized and was assumed to always be 20 bytes. Implementers of this ACP must take care to continue parsing old transactions correctly.

This ACP could be implemented by adding a new tx type that requires Ed25519 NodeIDs only. If the implementer chooses to do this, a separate follow-up ACP must be submitted detailing the format of that transaction.

### Future Work

In the future, usage of non-Ed25519 TLS certificates should be prohibited to remove any dependency on them. This will further secure the Avalanche network by reducing complexity. The path to doing so is not outlined in this ACP.

## Backwards Compatibility

An implementation of this proposal should not introduce any backwards compatibility issues. NodeIDs that are 20 bytes should continue to be treated as hashes of TLS certificates. NodeIDs of 32 bytes (size of Ed25519 public key) should be supported following implementation of this proposal.

## Reference Implementation

TLS certificate generation using an Ed25519 private key is standard. The golang standard library has a reference [implementation](https://github.com/golang/go/blob/go1.20.10/src/crypto/tls/generate_cert.go).

Parsing TLS certificates and extracting the public key is also standard. AvalancheGo already contains [code](https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/staking/verify.go#L55-L65) to verify the public key from a TLS certificate.

## Security Considerations

### Validation Criteria

Although Ed25519 is standardized in [RFC8032](https://datatracker.ietf.org/doc/html/rfc8032), it does not define strict validation criteria. This has led to inconsistencies in the validation criteria across implementations of the signature scheme. This is unacceptable for any protocol that requires participants to reach consensus on signature validity. Henry de Valance highlights the complexity of this issue [here](https://hdevalence.ca/blog/2020-10-04-its-25519am). 

From [Chalkias et al. 2020](https://eprint.iacr.org/2020/1244.pdf):

* The RFC 8032 and the NIST FIPS186-5 draft both require to reject non-canonically encoded points, but not all of the implementations follow those guidelines.
* The RFC 8032 allows optionality between using a permissive verification equation and a more strict verification equation. Different implementations use different equations meaning validation results can vary even across implementations that follow RFC 8032.

Zcash adopted [ZIP-215](https://zips.z.cash/zip-0215) (proposed by Henry de Valance) to explicitly define the Ed25519 validation criteria. Implementers of this ACP _*must*_ use the ZIP-215 validation criteria.

The [`ed25519consensus`](https://github.com/hdevalence/ed25519consensus) golang library is a minimal fork of golang's `crypto/ed25519` package with support for ZIP-215 verification. It is maintained by [Filippo Valsorda](https://github.com/FiloSottile) who also maintains many golang stdlib cryptography packages. It is strongly recommended to use this library for golang implementations.

## Open Questions

_Can this Ed25519 key be used in alternative communication protocols?_

Yes. Ed25519 can be used for alternative communication protocols like [QUIC](https://datatracker.ietf.org/group/quic/about) or [NOISE](http://www.noiseprotocol.org/noise.html). This ACP removes the reliance on TLS certificates and associates a Ed25519 public key with NodeIDs. This allows for experimentation with different communication protocols that may be better suited for a high throughput blockchain like Avalanche.

_Can this Ed25519 key be used for Verifiable Random Functions?_

Yes. VRFs, as specified in [RFC9381](https://datatracker.ietf.org/doc/html/rfc9381), can be constructed using elliptic curves that are secure in the cryptographic random oracle model. Ed25519 test vectors are provided in the RFC for implementers of an Elliptic Curve VRF (ECVRF). This allows for Avalanche validators to generate a VRF per block using their associated Ed25519 keys, including for Subnets.

## Acknowledgements

Thanks to [@StephenButtolph](https://github.com/StephenButtolph) and [@patrick-ogrady](https://github.com/patrick-ogrady) for their feedback on these ideas.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
