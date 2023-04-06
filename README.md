# libes
![Crates.io](https://img.shields.io/crates/l/libes?style=for-the-badge)
[![GitHub last commit](https://img.shields.io/github/last-commit/TJRoh01/libes?style=for-the-badge)](https://github.com/TJRoh01/libes)
[![Crates.io](https://img.shields.io/crates/v/libes?style=for-the-badge)](https://crates.io/crates/libes)
[![docs.rs](https://img.shields.io/docsrs/libes/latest?style=for-the-badge)](https://docs.rs/libes/latest/libes)
[![Libraries.io](https://img.shields.io/librariesio/release/cargo/libes?style=for-the-badge)](https://libraries.io/cargo/libes)

**lib**rary of **e**ncryption **s**cheme(s) is a collection of ECIES variants.

The goal of this is library is to become a one-stop shop for everything ECIES.

## Why use libes?
The rust cryptography ecosystem is swarming with crates, with varying degrees
of quality and documentation. I have taken it onto myself to navigate this,
and I want to share my findings with those who are trying to make sense of it like me.

In doing this I commit myself to:
- Maintaining a curated selection of relevant crates
    - Verifying that dependencies have not made mistakes in their implementations
    - Using dependencies with good performance and a high quality of code and documentation
- Providing a uniform and predictable API
    - Using shared constructors in the API to guarantee uniformity
    - Guaranteeing long-term support for all major releases
    - Striving for a high degree of clarity and detail in the documentation
- Keeping the library up to date & vulnerability-free
    - Automatically updating dependencies and testing code
    - Prioritizing issues & feedback on implementations

# Table of Contents
<!-- TOC -->
* [libes](#libes)
  * [Why use libes?](#why-use-libes)
* [Table of Contents](#table-of-contents)
* [FAQ](#faq)
* [About](#about)
  * [What is ECIES?](#what-is-ecies)
  * [ECIES Variants](#ecies-variants)
  * [ECIES-MAC Flowchart](#ecies-mac-flowchart)
  * [ECIES-AEAD Flowchart](#ecies-aead-flowchart)
  * [ECIES-SYN Flowchart](#ecies-syn-flowchart)
  * [Conditional Compilation](#conditional-compilation)
* [Encryption Scheme Support](#encryption-scheme-support)
  * [Support icon legend](#support-icon-legend)
  * [Elliptic Curve Support Matrix](#elliptic-curve-support-matrix)
  * [Encryption Support Matrix](#encryption-support-matrix)
  * [MAC Support Matrix](#mac-support-matrix)
* [License](#license)
* [Contributing](#contributing)
<!-- TOC -->

# FAQ
TBD

# About
## What is ECIES?
ECIES stands for **E**lliptic **C**urve **I**ntegrated **E**ncryption **S**cheme.
It is a type of cryptographic procedure which allows encrypting data
for a specific recipient given only the data to be encrypted and
the recipients public key, everything else is derived from the input
or generated with a
CSPRNG (**C**ryptographically **S**ecure **P**seudo-**R**andom **N**umber **G**enerator).

[Wikipedia](https://en.wikipedia.org/wiki/Integrated_Encryption_Scheme)  
[Crypto++](https://www.cryptopp.com/wiki/Elliptic_Curve_Integrated_Encryption_Scheme)  
[Practical Cryptography for Developers](https://cryptobook.nakov.com/asymmetric-key-ciphers/ecies-public-key-encryption)

## ECIES Variants
Cryptographic algorithms have evolved over time, and thus have grown into
two distinct ECIES variants as of writing.

Originally, ECIES relied on a key exchange operation, an encryption operation,
and a separate MAC operation.

A MAC (**M**essage **A**uthentication **C**ode) is necessary to provide
**Authenticity** on top of **Confidentiality**. By exploiting vulnerabilities
and/or compromised parameters, encrypted data could potentially be manipulated
to produce a desired output, other than what the sender intended. A MAC can be
used separately from the encrypted data to verify that such manipulation did 
not take place.

More recently adopted encryption algorithms like AES-GCM and ChaCha20-Poly1305
are AEAD (**A**uthenticated **E**ncryption with **A**dditional **D**ata) algorithms
which in addition to a ciphertext, also produce an Authentication Tag which serves
the same purpose that a MAC does in this case, but is integrated in the encryption
algorithm itself.

The library and documentation will refer to these two variants as:
- **ECIES-MAC** (Encryption with MAC)
- **ECIES-AEAD** (AEAD Encryption instead of MAC)

Iterating further on ECIES-AEAD, it could be further integrated by **synthesizing**
the IV/Nonce rather than **generating** it randomly. This would eliminate the need
to store & transmit the Nonce, as well as reduce the overhead by one or
two dozen bytes. Because there is already random data in the ephemeral key,
the risk of deriving the same encryption key twice is minimal, and thus it
should be safe to do so. This third variant will be referred to as **ECIES-SYN**.

**DISCLAIMER:** ECIES-SYN is my own idea, which I will only implement for
algorithms that I have done extensive research on to ensure that it is
cryptographically secure to do so. Regardless, I am not a cryptography
researcher and I can not give a guarantee that issues will not arise
in the future. If ECIES-SYN turns out to be useful/popular and resources allow,
I will make sure that it receives a security audit.

## ECIES-MAC Flowchart
Flowchart color coding:
- Red: Encryption
- Green: Elliptic Curve
- Blue: MAC
```mermaid
graph TB
    subgraph Input
        RK(Recipient's Public Key)
        MSG(Plaintext)
    end
    
    subgraph Generate
        CSPRNG(CSPRNG)
        EK_PUB(Ephemeral Public Key)
        EK_PRI(Ephemeral Private Key)
        IV(IV/Nonce)
    end
    
    subgraph Derive
        DER_FN("Derive Shared Secret</br>(e.g. with ECDH -> SHA512)</br><- and split ->")
        DER_ENC_KEY("Derived Encryption Key")
        DER_MAC_KEY("Derived MAC Key")
    end
    
    subgraph Process
        ENC_FN("Encrypt</br>(e.g. with AES-CBC)")
        MAC_FN("MAC</br>(e.g. with HMAC-SHA256)")
    end
    
    subgraph Output
        ENC_OUT(Ciphertext)
        IV_OUT(IV/Nonce)
        EK_OUT(Ephemeral Public Key)
        MAC_OUT(MAC)
    end

    CSPRNG --> IV & EK_PRI
    EK_PRI --> EK_PUB
    EK_PUB --> EK_OUT & MAC_FN
    IV --> IV_OUT & MAC_FN & ENC_FN
    RK --> DER_FN
    EK_PRI --> DER_FN
    DER_FN --> DER_ENC_KEY & DER_MAC_KEY
    DER_ENC_KEY --> ENC_FN
    DER_MAC_KEY --> MAC_FN
    MSG --> ENC_FN
    ENC_FN --> MAC_FN & ENC_OUT
    MAC_FN --> MAC_OUT
    
    %% Elliptic Curve Key link colorization
    linkStyle 2 stroke:#98fb98,stroke-width:2px;
    linkStyle 3 stroke:#98fb98,stroke-width:2px;
    linkStyle 8 stroke:#98fb98,stroke-width:2px;
    linkStyle 9 stroke:#98fb98,stroke-width:2px;
    
    %% MAC link colorization
    linkStyle 4 stroke:#0096ff,stroke-width:2px;
    linkStyle 6 stroke:#0096ff,stroke-width:2px;
    linkStyle 11 stroke:#0096ff,stroke-width:2px;
    linkStyle 13 stroke:#0096ff,stroke-width:2px;
    linkStyle 15 stroke:#0096ff,stroke-width:2px;
    linkStyle 17 stroke:#0096ff,stroke-width:2px;
    
    %% Encryption link colorization
    linkStyle 7 stroke:#f88379,stroke-width:2px;
    linkStyle 10 stroke:#f88379,stroke-width:2px;
    linkStyle 12 stroke:#f88379,stroke-width:2px;
    linkStyle 14 stroke:#f88379,stroke-width:2px;
    linkStyle 16 stroke:#f88379,stroke-width:2px;
```

## ECIES-AEAD Flowchart
Flowchart color coding:
- Red: Encryption
- Green: Elliptic Curve
- Blue: IV/Nonce
```mermaid
graph TB
    subgraph Input
        RK(Recipient's Public Key)
        MSG(Plaintext)
    end
    
    subgraph Generate
        CSPRNG(CSPRNG)
        EK_PUB(Ephemeral Public Key)
        EK_PRI(Ephemeral Private Key)
        IV(IV/Nonce)
    end
    
    subgraph Derive
        DER_FN("Derive Shared Secret</br>(e.g. with ECDH -> SHA256)")
        DER_ENC_KEY("Derived Encryption Key")
    end
    
    subgraph Process
        ENC_FN("Encrypt using AEAD</br>(e.g. with AES-GCM)")
    end
    
    subgraph Output
        ENC_OUT(Ciphertext)
        IV_OUT(IV/Nonce)
        EK_OUT(Ephemeral Public Key)
    end
    
    CSPRNG --> EK_PRI & IV
    EK_PRI --> EK_PUB
    EK_PUB --> EK_OUT
    RK --> DER_FN
    EK_PRI --> DER_FN
    DER_FN --> DER_ENC_KEY
    DER_ENC_KEY --> ENC_FN
    MSG --> ENC_FN
    ENC_FN --> ENC_OUT
    IV --> ENC_FN & IV_OUT
    
    %% Elliptic Curve Key link colorization
    linkStyle 2 stroke:#98fb98,stroke-width:2px;
    linkStyle 3 stroke:#98fb98,stroke-width:2px;
    linkStyle 4 stroke:#98fb98,stroke-width:2px;
    linkStyle 5 stroke:#98fb98,stroke-width:2px;
    
    %% Nonce link colorization
    linkStyle 10 stroke:#0096ff,stroke-width:2px;
    linkStyle 11 stroke:#0096ff,stroke-width:2px;
    
    %% Encryption link colorization
    linkStyle 6 stroke:#f88379,stroke-width:2px;
    linkStyle 7 stroke:#f88379,stroke-width:2px;
    linkStyle 8 stroke:#f88379,stroke-width:2px;
    linkStyle 9 stroke:#f88379,stroke-width:2px;
```

## ECIES-SYN Flowchart
Flowchart color coding:
- Red: Encryption
- Green: Elliptic Curve
- Blue: IV/Nonce
```mermaid
graph TB
    subgraph Input
        RK(Recipient's Public Key)
        MSG(Plaintext)
    end
    
    subgraph Generate
        CSPRNG(CSPRNG)
        EK_PUB(Ephemeral Public Key)
        EK_PRI(Ephemeral Private Key)
    end
    
    subgraph Derive
        DER_FN("Derive Shared Secret</br>(e.g. with ECDH -> SHA512)</br><- and split ->")
        DER_IV("Derived IV/Nonce")
        DER_ENC_KEY("Derived Encryption Key")
    end
    
    subgraph Process
        ENC_FN("Encrypt using AEAD</br>(e.g. with AES-GCM)")
    end
    
    subgraph Output
        ENC_OUT(Ciphertext)
        EK_OUT(Ephemeral Public Key)
    end
    
    CSPRNG --> EK_PRI
    EK_PRI --> EK_PUB
    EK_PUB --> EK_OUT
    RK --> DER_FN
    EK_PRI --> DER_FN
    DER_FN --> DER_ENC_KEY & DER_IV
    DER_IV --> ENC_FN
    DER_ENC_KEY --> ENC_FN
    MSG --> ENC_FN
    ENC_FN --> ENC_OUT
    
    %% Elliptic Curve Key link colorization
    linkStyle 1 stroke:#98fb98,stroke-width:2px;
    linkStyle 2 stroke:#98fb98,stroke-width:2px;
    linkStyle 3 stroke:#98fb98,stroke-width:2px;
    linkStyle 4 stroke:#98fb98,stroke-width:2px;
    
    %% Nonce link colorization
    linkStyle 5 stroke:#0096ff,stroke-width:2px;
    linkStyle 8 stroke:#0096ff,stroke-width:2px;
    
    %% Encryption link colorization
    linkStyle 6 stroke:#f88379,stroke-width:2px;
    linkStyle 7 stroke:#f88379,stroke-width:2px;
    linkStyle 9 stroke:#f88379,stroke-width:2px;
    linkStyle 10 stroke:#f88379,stroke-width:2px;
```

## Conditional Compilation
All algorithm combinations are gated behind features, to reduce how much is
being compiled. Features are named exactly like the algorithm names in the
support matrices (if there are alternative names like P-521 and secp521r1 then
they are aliases, so you can enable either). There are also no ECIES methods
hard-defined, the library relies on a type alias being defined, and then the 
appropriate traits will automatically implement on it,
exposing high-level functionality.

**NOTE:** No ECIES variants are available without activating any features,
at minimum one of each feature categories must be activated:
- Elliptic Curve Key (e.g. x25519)
- Encryption (e.g. AES-GCM)
- Variant (e.g. ECIES-AEAD)

Additionally, a MAC feature (e.g. HMAC-SHA256) can be activated to enable the
use of ECIES-MAC.

# Encryption Scheme Support
## Support icon legend
- 🚀 Completed
- 🏗️ Development
- 📅 Planned
- 🤔 Planning
- 🚫 Can/Will not implement

## Elliptic Curve Support Matrix
|     Algorithm     | ECIES-MAC | ECIES-AEAD | ECIES-SYN |
|:-----------------:|:---------:|:----------:|:---------:|
|      x25519       |    🏗️    |    🏗️     |    📅     |
|      ed25519      |    🏗️    |    🏗️     |    📅     |
| P-256 / secp256r1 |    🤔     |     🤔     |    🤔     |
|  P-384 secp384r1  |    🤔     |     🤔     |    🤔     |
| P-521 / secp521r1 |    🤔     |     🤔     |    🤔     |

## Encryption Support Matrix
|     Algorithm      | ECIES-MAC | ECIES-AEAD | ECIES-SYN |
|:------------------:|:---------:|:----------:|:---------:|
| ChaCha20-Poly1305  |  🚫[^1]   |   🚫[^1]   |  🚫[^1]   |
| XChaCha20-Poly1305 |    🏗️    |    🏗️     |    📅     |
|      AES-GCM       |    🤔     |     🤔     |    🤔     |

## MAC Support Matrix
|  Algorithm  | ECIES-MAC |
|:-----------:|:---------:|
| HMAC-SHA256 |    🏗️    |
| HMAC-SHA512 |    🤔     |

[^1]: ChaCha20 uses a 96-bit nonce,
which when generated using a random function has an unsatisfactory
risk of collision. XChaCha20 uses a 192-bit nonce
where that is no longer an issue.

# License
Licensed under either of:
- [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
- [MIT license](http://opensource.org/licenses/MIT)

at your option.

# Contributing
All contributions are very appreciated. If you spot a mistake or a vulnerability in
this crate or any of its dependencies please open an issue. Currently, there is no
template for issues or pull requests, but please try to include enough information
to be able to determine what to do without having to ask too many follow-up questions.

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this project by you, as defined in the Apache-2.0 license,
shall be dual licensed as above , without any additional terms or conditions.
