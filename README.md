# BIP32-Ed25519-Swift

A Swift implementation of ARC-0052 Algorand, in accordance with the paper BIP32-Ed25519 Hierarchical Deterministic Keys over a Non-linear Keyspace.

Note that this library has NOT undergone audit and is NOT recommended for production use.

## Git Hooks

This repo comes with git hooks.

Copy the hooks under git-hooks into the .git/hooks/ directory:

```
cp git-hooks/* .git/hooks/
```

## How to Use

To initialize a wallet (using MnemmonicSwift for BIP-39 support) from a seed phrase:

```swift
import bip32_ed25519_swift
import MnemonicSwift
let seed = try Mnemonic.deterministicSeedString(from: "salon zoo engage submit smile frost later decide wing sight chaos renew lizard rely canal coral scene hobby scare step bus leaf tobacco slice")
let c = Bip32Ed25519(seed: seed)
```

Now you can generate keys using a BIP-44 derivation path:

```swift
let pk = c.keyGen(context: KeyContext.Address, account: 0, change: 0, keyIndex: 0)
```

To sign an Algorand transaction, you can use the signAlgoTransaction.

```swift
let prefixEncodedTx = Data(base64Encoded: "VFiJo2FtdM0D6KNmZWXNA+iiZnbOAkeSd6NnZW6sdGVzdG5ldC12MS4womdoxCBIY7UYpLPITsgQ8i1PEIHLD3HwWaesIN7GL39w5Qk6IqJsds4CR5Zfo3JjdsQgYv6DK3rRBUS+gzemcENeUGSuSmbne9eJCXZbRrV2pvOjc25kxCBi/oMretEFRL6DN6ZwQ15QZK5KZud714kJdltGtXam86R0eXBlo3BheQ==")
let sig = c.signAlgoTransaction(context: KeyContext.Address, account: 0, change: 0, keyIndex: 0, prefixEncodedTx: prefixEncodedTx)
```

Where prefixEncodedTx is a transaction that has been compiled with the SDK's transaction builder. The signature returned can be verified against the public key:

```swift
let pk = c.keyGen(context: KeyContext.Address, account: 0, change: 0, keyIndex: 0)
let result = c.verifyWithPublicKey(signature: sig, message: prefixEncodedTx, publicKey: pk)
```

It is also possible to sign arbitrary data. You need to specify a JSON scehma and encoding type (none, base64, msgpack).

For example (with schema under "schemas/auth.request.json" in the Tests folder):

```swift
let schema = Schema("auth.request.json")
let challengeJSON = """
    {
        "0": 28, "1": 103, "2": 26, "3": 222, "4": 7, "5": 86, "6": 55, "7": 95,
        "8": 197, "9": 179, "10": 249, "11": 252, "12": 232, "13": 252, "14": 176,
        "15": 39, "16": 112, "17": 131, "18": 52, "19": 63, "20": 212, "21": 58,
        "22": 226, "23": 89, "24": 64, "25": 94, "26": 23, "27": 91, "28": 128,
        "29": 143, "30": 123, "31": 27
    }
    """
let sig = c.signData(context: KeyContext.Address, account: 0, change: 0, keyIndex: 0, data: data, metadata: SignMetadata(encoding: Encoding.none, schema: schema))
let result = c.verifyWithPublicKey(signature: sig, message: Data(challengeJSON.utf8), publicKey: pk)
```

or using base64:

```swift
let challengeJSONB64 = "eyIwIjogMjgsICIxIjogMTAzLCAiMiI6IDI2LCAiMyI6IDIyMiwgIjQiOiA3LCAiNSI6IDg2LCAiNiI6IDU1LCAiNyI6IDk1LCAiOCI6IDE5NywgIjkiOiAxNzksICIxMCI6IDI0OSwgIjExIjogMjUyLCAiMTIiOiAyMzIsICIxMyI6IDI1MiwgIjE0IjogMTc2LCAiMTUiOiAzOSwgIjE2IjogMTEyLCAiMTciOiAxMzEsICIxOCI6IDUyLCAiMTkiOiA2MywgIjIwIjogMjEyLCAiMjEiOiA1OCwiMjIiOiAyMjYsICIyMyI6IDg5LCAiMjQiOiA2NCwgIjI1IjogOTQsICIyNiI6IDIzLCAiMjciOiA5MSwgIjI4IjogMTI4LCAiMjkiOiAxNDMsICIzMCI6IDEyMywgIjMxIjogMjd9"
let sig = c.signData(context: KeyContext.Address, account: 0, change: 0, keyIndex: 0, data: data: Data(challengeJSONB64.utf8), metadata: SignMetadata(encoding: Encoding.base64, schema: schema))
let result = c.verifyWithPublicKey(signature: sig, message: Data(challengeJSONB64.utf8), publicKey: pk)
```

You can generate a shared secret with someone using ECDH. They will need to provide you with their Ed25519 public key, as provided by keyGen. You will also need to agree on an "order" of whose public key will be concatenated first and whose second.

```swift
let sharedSecret = c.ECDH(context: KeyContext.Identity, account: 0, change: 0, keyIndex: 0, otherPartyPub: otherPubKey, meFirst: true)
```

Note that under the hood the sharedSecret is calculated using x25519 form.

### Deriving Child Public Keys

You can also utilize `deriveKey` to derive extended public keys thus allowing `deriveChildNodePublic` to softly derive `N` public keys / addresses using a single extended key / root.

> [!IMPORTANT]
> We distinguish between our 32 byte public key (pk) and the 64 byte extended public key (xpk) where xpk is used to derive child nodes in `deriveChildNodePublic` & `deriveChildNodePrivate`

```swift
let bip44Path: [UInt32] = [c.harden(44), c.harden(283), c.harden(0), 0]

let walletRoot = c.deriveKey(
    rootKey: c.fromSeed(data),
    bip44Path: bip44Path,
    isPrivate: false,
    derivationType: BIP32DerivationType.Peikert
)
```

The output of `deriveKey` is an xpk which can be shared & using the following
```swift
let derivedKey = try c.deriveChildNodePublic(extendedKey: walletRoot, index: UInt32(i), g: BIP32DerivationType.Peikert)
```

can derive keys at the index specified.

## Formatting and Linting

This repo comes with both [Swiftformat](https://github.com/nicklockwood/SwiftFormat) and [SwiftLint](https://github.com/realm/SwiftLint).

SwiftLint comes with its own `SwiftLintBuildToolPlugin` and is such run as part of any normal `swift build` call.

SwiftFormat is run as part of the pre-commit hook, and can otherwise be called from the command line with `swift package plugin --allow-writing-to-package-directory swiftformat .`. As this also triggers a `swift build`, SwiftLint gets run as well.

The two do overlap but, generally speaking, SwiftFormat formats the code while SwiftLint warns or errors out on code-smells.

## Requirements

The Package.swift file specifies the minimum version of Swift required to build this library, as well as the minimum versions of the platforms. `.swift-version` also specifies the version for linting.

## License

Copyright 2024 Algorand Foundation

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
