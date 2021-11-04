# RFC #38: Output Types for Tokenization and Smart Contracts

- Feature name: `output_types_for_tokenization_and_sc`
- Start date: 2021-11-04
- RFC PR:

# Summary

This document proposes new output types and transaction validation rules for the IOTA protocol to support **native
tokenization** and **smart contract** features.

Native tokenization refers to the capability of the IOTA ledger to **track the ownership and transfer of user defined
tokens, so-called native tokens**, thus making it a **multi-asset ledger**. The scalable and feeless nature of IOTA
makes it a prime candidate for tokenization use cases.

The **IOTA Smart Contract Protocol (ISCP)** is a **layer 2 extension** of the IOTA protocol that adds smart contract
features to the Tangle. Many so-called **smart contract chains**, which anchor their state to the base ledger, can be
run in parallel. Users wishing to interact with smart contract chains can send **requests to layer 1 chain accounts
either as regular transactions or directly to the chain**, but **chains may also interact with other chains** in a
trustless manner through the Tangle.

This RFC presents output types that realize the required new features:
- Smart contract chains have a new account type, called alias account, represented by an **alias output.**
- Requests to smart contract chains can be carried out using the configurable new output type called
  **extended output.**
- Native tokens have their own **supply control policy** enforced by **foundry outputs.**
- Layer 1 native **non-fungible tokens** (unique tokens with attached metadata) are introduced via **NFT outputs.**

# Motivation

IOTA transitioned from an account based ledger model to an unspent transaction output (UTXO) model with the upgrade to
[Chrysalis phase 2](https://github.com/iotaledger/protocol-rfcs/pull/18). In this model, transactions explicitly
reference funds produced by previous transactions to be consumed. This property is desired for scalability: transaction
validation does not depend on the shared global state and, as such, transactions can be validated in parallel.
Double-spends can easily be detected as they spend the very same output more than once.

The UTXO model becomes even more powerful when unlocking criteria (validation) of outputs is extended as demonstrated
by the [EUTXO model (Chakravarty et al., 2020)](https://fc20.ifca.ai/wtsc/WTSC2020/WTSC20_paper_25.pdf): instead of
requiring only a valid signature for the output's address to unlock it, additional unlocking conditions can be
programmed into outputs. This programmability of outputs is the main idea behind the new output types presented in this
document.

Today, outputs in the IOTA protocol are designed for one specific use case: the single asset cryptocurrency. The aim of
this RFC is to design several output types for the use cases of:
- Native Tokenization Framework,
- ISCP style smart contracts,
- seamless interoperability between layer 1 and layer 2 tokenization concepts.

Users will be able to mint their own native tokens directly in the base ledger, which can then be transferred without
any fees just like regular IOTA coins. Each native token has its own supply control policy enforced by the protocol.
These policies are transparent to all network participants. Issuers will be able to store metadata about their tokens
on-ledger, accessible to anyone.

Non-fungible tokens can be minted and transferred with zero fees. The validated issuers of such NFTs are immutably
attached to the tokens, making it impossible to counterfeit them.

Users will be able to interact with smart contracts by posting requests through the Tangle. Requests can carry commands
to smart contracts and can additionally also transfer native tokens and NFTs. By depositing native tokens to smart
contracts, their features can be greatly enhanced and programmed to specific use cases.

The proposal in this RFC not only makes it possible to transfer native tokens to layer 2 smart contracts, but tokens
that originate from layer 2 smart contract chains can also be wrapped into their respective layer 1 representation.
Smart contract chains may transfer tokens between themselves through this mechanism, and they can also post requests to
other chains.

Composability of smart contracts extends the realm of one smart contract chain, as smart contracts residing on
different chains can call each other in a trustless manner.

In conclusion, the IOTA protocol will become a scalable general purpose multi-asset DLT with the addition of smart
contracts and native tokenization frameworks. The transition is motivated by the ever-growing need for a scalable and
affordable decentralized application platform.

# Detailed Design

Outputs in the UTXO model are essential, core parts of the protocol. The new output types introduce new validation and
unlocking mechanisms, therefore the protocol needs to be adapted. The structure of the remaining sections is as follows:
1. [Introduction to ledger programmability](#ledger-programmability)
2. [Data types, subschemas and protocol constants](#data-types)
3. [Transaction Payload changes compared to Chrysalis Part 2](#transaction-payload-changes)
4. [New concepts of output design](#new-concepts)
    - [Native tokens](#native-tokens-in-outputs)
    - [Optional output features](#optional-output-features)
    - [Chain constraint](#chain-constraint-in-utxo)
6. [Detailed design of new output types](#output-design)
7. [New unlocking mechanisms](#unlocking-chain-script-locked-outputs)
8. [Discussion](#drawbacks)

## Ledger Programmability

The current UTXO model only provides support to transfer IOTA coins. However, the UTXO model presents a unique
opportunity to extend the range of possible applications by programming outputs.

Programming the base ledger of a DLT is not a new concept. Bitcoin uses the UTXO model and attaches small executables
(scripts) that need to be executed during transaction validation. The bitcoin script language is however not
[Turing-complete](https://en.wikipedia.org/wiki/Turing_completeness) as it can only support a small set of instructions
that are executed in a stack based environment. As each validator has to execute the same scripts and arrive at the
same conclusion, such scripts must terminate very quickly. Also, as transaction validation happens in the context of
the transaction and block, the scripts have no access to the global shared state of the system (all unspent transaction
outputs).

The novelty of Ethereum was to achieve quasi Turing-completeness by employing an account based model and gas to limit
resource usage during program execution. As the amount of gas used per block is limited, only quasi Turing-completeness
can be achieved. The account based model of Ethereum makes it possible for transactions to have access to the global
shared state of the system, furthermore, transactions are executed one-after-the-other. These two properties make
Ethereum less scalable and susceptible to high transaction fees.

Cardano achieves UTXO programmability by using the EUTXO model. This makes it possible to represent smart contracts in
a UTXO model as state machines. In EUTXO, states of the machine are encoded in outputs, while state transition rules
are governed by scripts. Just like in bitcoin, these scripts may only use a limited set of instructions.

It would be quite straightforward to support EUTXO in IOTA too, except that IOTA transactions are feeless. There is no
reward to be paid out to validators for validating transactions, as all nodes in the network validate all transactions.
Due to the unique data structure of the Tangle, there is no need for miners to explicitly choose which transactions are
included in the ledger, but there still has to be a notion of objective validity of transactions. Since it is not
possible without fees to penalize scripts that consume excessive network resources (node CPU cycles) during transaction
validation, IOTA has to be overly restrictive about what instructions are supported on layer 1.

It must also be noted that UTXO scripts are finite state machines with the state space restricted by the output and
transaction validation rules. It makes expressiveness of UTXO scripts inherently limited. In the context of complicated
application logic required by use cases such as modern DeFi, this leads to unconventional and complicated architectures
of the application, consisting of many interacting finite state machines. Apart from complexity and UX costs, it also
has performance and scalability penalties.

For the reason mentioned above, **IOTA chooses to support configurable yet hard-coded scripts for output and
transaction validation on layer 1.** The general full-scale quasi Turing-complete programmability of the IOTA ledger is
achieved by extending the ledger state transition function with layer 2 smart contract chains. This not only makes it
possible to keep layer 1 scalable and feeless, but also allows to support any type of virtual machine on layer 2 to
program advanced business logic and features.

Below, several new output types are discussed that implement their own configurable script logic. They can be viewed as
UTXO state machines in which the state of the machine is encoded as data inside the output. The state transition rules
are defined by the output type and by the parameters chosen upon deployment.

## Data Types

All data types and their definitions that may be used throughout this document:

| Name   | Description   |
| ------ | ------------- |
| uint8  | An unsigned 8 bits integer encoded in Little Endian. |
| uint16  | An unsigned 16 bits integer encoded in Little Endian. |
| uint32  | An unsigned 32 bits integer encoded in Little Endian. |
| uint64  | An unsigned 64 bits integer encoded in Little Endian. |
| uint256  | An unsigned 256 bits integer encoded in Little Endian. |
| ByteArray[N] | A static size byte array of size N.   |
| ByteArray | A dynamically sized byte array. Prepended with uint32 byte length. |

## Subschema Notation

The following table describes subschema notations:

<table>
    <tr>
        <th>Name</th>
        <th>Description</th>
    </tr>
    <tr>
        <td><code>oneOf</code></td>
        <td>One of the listed subschemas.</td>
    </tr>
    <tr>
        <td><code>optOneOf</code></td>
        <td>Optionally one of the listed subschemas.</td>
    </tr>
    <tr>
        <td><code>anyOf</code></td>
        <td>Any (one or more) of the listed subschemas.</td>
    </tr>
    <tr>
        <td><code>optAnyOf</code></td>
        <td>Optionally any (one or more) of the listed subschemas.</td>
    </tr>
    <tr>
        <td><code>atMostOneOfEach</code></td>
        <td>At most one of each of the listed subschemas.</td>
    </tr>
</table>

## Global Protocol Parameters

| Name   | Type | Value | Description |
| ------ | ---- | ----- | ----------- |
| Minimum Dust Deposit | uint64 | TBD | Amount of IOTA coins that need to be present in a `SimpleOutput` not to be considered dust. |
| Max Native Token Count Per Output | uint16 | 256 | Maximum possible number of different native tokens that can reside in one output. |
| Max Indexation Tag Length | uint16 | 64 | Maximum possible length in bytes of an `Indexation Tag`. |
| Max Metadata Length | uint32 | TBD | Maximum possible length in bytes of a `Metadata` field. |
| Max Inputs Count | uint16 | 127 | Maximum possible count of inputs consumed in a transaction. |
| Max Outputs Count | uint16 | 127 | Maximum possible count of outputs created in a transaction. |


## Transaction Payload Changes

The new output types and unlocking mechanisms require new transaction validation rules, furthermore some protocol rules
have been modified compared to
[Chrysalis Part 2 Transaction Payload RFC.](https://github.com/luca-moser/protocol-rfcs/blob/signed-tx-payload/text/0000-transaction-payload/0000-transaction-payload.md)

<i>Transaction Payload</i> has the following structure:

### Serialized Layout

A <i>Transaction Payload</i> is made up of two parts:
1. The <i>Transaction Essence</i> part which contains the inputs, outputs and an optional embedded payload.
2. The <i>Unlock Blocks</i> which unlock the <i>Transaction Essence</i>'s inputs. In case the unlock block contains a
3. signature, it signs the Blake2b-256 hash of the serialized <i>Transaction Essence</i> part.

All values are serialized in little-endian encoding. In contrast to the current IOTA protocol inputs and outputs are
encoded as lists, which means that they can contain duplicates and may not be sorted.

A [Blake2b-256](https://tools.ietf.org/html/rfc7693) hash of the entire serialized data makes up 
<i>Transaction Payload</i>'s ID.

Following table structure describes the entirety of a <i>Transaction Payload</i>'s serialized form. New output types
and unlock blocks will be discussed later in detail, but they are mentioned in the payload structure to help the reader
understand their context:

<p></p>

<table>
    <tr>
        <th>Name</th>
        <th>Type</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>Payload Type</td>
        <td>uint32</td>
        <td>
        Set to <strong>value 0</strong> to denote a <i>Transaction Payload</i>.
        </td>
    </tr>
    <tr>
        <td valign="top">Essence <code>oneOf</code></td>
        <td colspan="2">
            <details open="true">
                <summary>Transaction Essence</summary>
                <blockquote>
                Describes the essence data making up a transaction by defining its inputs and outputs and an optional payload.
                </blockquote>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Transaction Type</td>
                        <td>uint8</td>
                        <td>
                        Set to <strong>value 0</strong> to denote a <i>Transaction Essence</i>.
                        </td>
                    </tr>
                   <tr>
                        <td>Inputs Count</td>
                        <td>uint16</td>
                        <td>The amount of inputs proceeding.</td>
                    </tr>
                   <tr>
                        <td valign="top">Inputs <code>anyOf</code></td>
                        <td colspan="2">
                            <details>
                                <summary>UTXO Input</summary>
                                <blockquote>
                                Describes an input which references an unspent transaction output to consume.
                                </blockquote>
                                <table>
                                    <tr>
                                        <td><b>Name</b></td>
                                        <td><b>Type</b></td>
                                        <td><b>Description</b></td>
                                    </tr>
                                    <tr>
                                        <td>Input Type</td>
                                        <td>uint8</td>
                                        <td>
                                            Set to <strong>value 0</strong> to denote an <i>UTXO Input</i>.
                                        </td>
                                    </tr>
                                    <tr>
                                        <td>Transaction ID</td>
                                        <td>Array&lt;byte&gt;[32]</td>
                                        <td>The BLAKE2b-256 hash of the transaction from which the UTXO comes from.</td>
                                    </tr>
                                    <tr>
                                        <td>Transaction Output Index</td>
                                        <td>uint16</td>
                                        <td>The index of the output on the referenced transaction to consume.</td>
                                    </tr>
                                </table>
                            </details>
                        </td>
                    </tr>
                   <tr>
                        <td>Outputs Count</td>
                        <td>uint16</td>
                        <td>The amount of outputs proceeding.</td>
                    </tr>
                   <tr>
                        <td valign="top">Outputs <code>anyOf</code></td>
                        <td colspan="2">
                            <details>
                                <summary>SimpleOutput</summary>
                                <blockquote>
                                Describes an IOTA coin deposit to a single address.
                                </blockquote>
                            </details>
                            <details>
                                <summary>ExtendedOutput</summary>
                                <blockquote>
                                Describes a deposit to a single address. The output might contain optional feature blocks and native tokens.
                                </blockquote>
                            </details>
                            <details>
                                <summary>AliasOutput</summary>
                                <blockquote>
                                Describes an alias account in the ledger.
                                </blockquote>
                            </details>
                            <details>
                                <summary>FoundryOutput</summary>
                                <blockquote>
                                Describes a foundry that controls supply of native tokens.
                                </blockquote>
                            </details>
                            <details>
                                <summary>NFTOutput</summary>
                                <blockquote>
                                Describes a unique, non-fungible token deposit to a single address.
                                </blockquote>
                            </details>
                        </td>
                    </tr>
                    <tr>
                        <td>Payload Length</td>
                        <td>uint32</td>
                        <td>The length in bytes of the optional payload.</td>
                    </tr>
                   <tr>
                        <td valign="top">Payload <code>optOneOf</code></td>
                        <td colspan="2">
                            <details>
                                <summary>Indexation Payload</summary>
                            </details>
                        </td>
                    </tr>
                </table>
            </details>
        </td>
    </tr>
    <tr>
        <td>Unlock Blocks Count</td>
        <td>uint16</td>
        <td>The count of unlock blocks proceeding. Must match count of inputs specified.</td>
    </tr>
    <tr>
        <td valign="top">Unlock Blocks <code>anyOf</code></td>
        <td colspan="2">
            <details>
                <summary>Signature Unlock Block</summary>
                <blockquote>
                Defines an unlock block containing signature(s) unlocking input(s).
                </blockquote>
                <table>
                    <tr>
                        <th>Name</th>
                        <th>Type</th>
                        <th>Description</th>
                    </tr>
                    <tr>
                        <td>Unlock Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 0</strong> to denote a <i>Signature Unlock Block</i>.
                        </td>
                    </tr>
                    <tr>
                        <td valign="top">Signature <code>oneOf</code></td>
                        <td colspan="2">
                             <details>
                                <summary>Ed25519 Signature</summary>
                                <table>
                                    <tr>
                                        <th>Name</th>
                                        <th>Type</th>
                                        <th>Description</th>
                                    </tr>
                                    <tr>
                                        <td>Signature Type</td>
                                        <td>uint8</td>
                                        <td>
                                            Set to <strong>value 0</strong> to denote an <i>Ed25519 Signature</i>.
                                        </td>
                                    </tr>
                                    <tr>
                                        <td>Public key</td>
                                        <td>ByteArray[32]</td>
                                        <td>The public key of the Ed25519 keypair which is used to verify the signature.</td>
                                    </tr>
                                    <tr>
                                        <td>Signature</td>
                                        <td>ByteArray[64]</td>
                                        <td>The signature signing the Blake2b-256 hash of the serialized <i>Transaction Essence</i>.</td>
                                    </tr>
                                </table>
                            </details>
                        </td>
                    </tr>
                </table>
            </details>
            <details>
                <summary>Reference Unlock Block</summary>
                <blockquote>
                References a previous unlock block in order to substitute the duplication of the same unlock block data for inputs which unlock through the same data.
                </blockquote>
                <table>
                    <tr>
                        <th>Name</th>
                        <th>Type</th>
                        <th>Description</th>
                    </tr>
                    <tr>
                        <td>Unlock Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 1</strong> to denote a <i>Reference Unlock Block</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Reference</td>
                        <td>uint16</td>
                        <td>Represents the index of a previous unlock block.</td>
                    </tr>
                </table>
            </details>
            <details>
                <summary>Alias Unlock Block</summary>
                <blockquote>
                Points to the unlock block of a consumed alias output.
                </blockquote>
                <table>
                    <tr>
                        <th>Name</th>
                        <th>Type</th>
                        <th>Description</th>
                    </tr>
                    <tr>
                        <td>Unlock Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 2</strong> to denote an <i>Alias Unlock Block</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Alias Reference Unlock Index</td>
                        <td>uint16</td>
                        <td>Index of input and unlock block corresponding to an alias output.</td>
                    </tr>
                </table>
            </details>
            <details>
                <summary>NFT Unlock Block</summary>
                <blockquote>
                Points to the unlock block of a consumed NFT output.
                </blockquote>
                <table>
                    <tr>
                        <th>Name</th>
                        <th>Type</th>
                        <th>Description</th>
                    </tr>
                    <tr>
                        <td>Unlock Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 3</strong> to denote a <i>NFT Unlock Block</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>NFT Reference Unlock Index</td>
                        <td>uint16</td>
                        <td>Index of input and unlock block corresponding to an NFT output.</td>
                    </tr>
                </table>
            </details>
        </td>
    </tr>
</table>

### Transaction Parts

In general, all parts of a <i>Transaction Payload</i> begin with a byte describing the type of the given part to keep
the flexibility to introduce new types/versions of the given part in the future.

#### Transaction Essence Data

As described, the <i>Transaction Essence</i> of a <i>Transaction Payload</i> carries the inputs, outputs, and an
optional payload. The <i>Transaction Essence</i> is an explicit type and therefore starts with its own
<i>Transaction Essence Type</i> byte which is of value 0.

A <i>Transaction Essence</i> must contain at least one input and output.

##### Inputs

The <i>Inputs</i> part holds the inputs to consume, respectively, to fund the outputs of the
<i>Transaction Essence</i>. There is only one type of input as of now, the <i>UTXO Input</i>. In the future, more types
of inputs may be specified as part of protocol upgrades.

Each defined input must be accompanied by a corresponding <i>Unlock Block</i> at the same index in the
<i>Unlock Blocks</i> part of the <i>Transaction Payload</i>.

If multiple inputs can be unlocked through the same <i>Unlock Block</i>, then the given <i>Unlock Block</i> only needs
to be specified at the index of the first input which gets unlocked by it.

Subsequent inputs which are unlocked through the same data must have a <i>Reference Unlock Block</i>,
<i>Alias Unlock Block</i> or <i>NFT Unlock Block</i> depending on the unlock mechanism, pointing to the index of a
previous <i>Unlock Block</i>. This ensures that no duplicate data needs to occur in the same transaction.

###### UTXO Input

An <i>UTXO Input</i> is an input which references an output of a previous transaction by using the given transaction's
BLAKE2b-256 hash + the index of the output on that transaction. An <i>UTXO Input</i> must be accompanied by an
<i>Unlock Block</i> for the corresponding type of output the <i>UTXO Input</i> is referencing.

Example: If the output which is referenced by an input is derived from an Ed25519 address, then the corresponding
unlock block must be of type <i>Signature Unlock Block</i> holding an Ed25519 signature.

##### Outputs

The <i>Outputs</i> part holds the outputs to create with this <i>Transaction Payload</i>.

###### SimpleOutput

Formerly known as <i>SigLockedSingleOutput</i>, the <i>SimpleOutput</i> defines an output (with a certain IOTA coins
amount) to a single target address which is unlocked via the correct unlock block given the type of the address. The
output can hold addresses of different types.

###### ExtendedOutput

An output to a single target address that may carry native tokens and optional feature blocks. Defined in
[Extended Output section.](#extended-output)

###### AliasOutput

An output that represents an alias account in the ledger. Defined in
[Alias Output section.](#alias-output)

###### FoundryOutput

An output that represents a token foundry in the ledger. Defined in
[Foundry Output section.](#foundry-output)

###### NFTOutput

An output that represents a non-fungible token in the ledger. Defined in
[NFT Output section.](#nft-output)

##### Payload

The payload part of a <i>Transaction Essence</i> can hold an optional payload. This payload does not affect the
validity of the <i>Transaction Essence</i>. If the transaction is not syntactically valid, then the payload must also
be discarded.

Supported payload types to be embedded into a <i>Transaction Essence</i>:

| Name                  | Type Value |
| --------------------- | ---------- |
| Indexation Payload    | 2          |

#### Unlock Blocks

The <i>Unlock Blocks</i> part holds the unlock blocks unlocking inputs within an <i>Transaction Essence</i>.

There are different types of <i>Unlock Blocks</i>:
<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Value</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Signature Unlock Block</td>
        <td>0</td>
        <td>An unlock block holding one or more signatures unlocking one or more inputs.</td>
    </tr>
    <tr>
        <td>Reference Unlock Block</td>
        <td>1</td>
        <td>An unlock block which must reference a previous unlock block which unlocks also the input at the same index as this <i>Reference Unlock Block</i>.</td>
    </tr>
    <tr>
        <td>Alias Unlock Block</td>
        <td>2</td>
        <td>An unlock block which must reference a previous unlock block which unlocks the alias that the input is locked to.</td>
    </tr>
    <tr>
        <td>NFT Unlock Block</td>
        <td>3</td>
        <td>An unlock block which must reference a previous unlock block which unlocks the NFT that the input is locked to.</td>
    </tr>
</table>

##### Signature Unlock Block

A <i>Signature Unlock Block</i> defines an <i>Unlock Block</i> which holds one or more signatures signing the
Blake2b-256 hash of the <i>Transaction Essence</i> (including the optional payload).

##### Reference Unlock Block

A <i>Reference Unlock Block</i> defines an <i>Unlock Block</i> which references a previous <i>Unlock Block</i> (which
must not be another <i>Reference Unlock Block</i>). It must be used if multiple inputs can be unlocked through the same
origin <i>Unlock Block</i>.

Example:
Consider an <i>Transaction Essence</i> containing <i>UTXO Inputs</i> A, B and C, where A and C are both spending the
UTXOs originating from the same Ed25519 address. The <i>Unlock Block</i> part must thereby have the following structure:

| Index | Must Contain                                                                                                     |
| ----- | ---------------------------------------------------------------------------------------------------------------- |
| 0     | A <i>Signature Unlock Block</i> holding the corresponding Ed25519 signature to unlock A and C.                   |
| 1     | A <i>Signature Unlock Block</i> which unlocks B.                                                                 |
| 2     | A <i>Reference Unlock Block</i> which references index 0, since C also gets unlocked by the same signature as A. |

##### Alias Unlock Block

An <i>Alias Unlock Block</i> defines an <i>Unlock Block</i> which references a previous <i>Unlock Block</i>
corresponding to the alias that the input is locked to. Defined in
[alias unlocking.](#alias-locking--unlocking)

##### NFT Unlock Block

An <i>NFT Unlock Block</i> defines an <i>Unlock Block</i> which references a previous <i>Unlock Block</i> corresponding
to the NFT that the input is locked to. Defined in
[NFT unlocking.](#nft-locking--unlocking)

### Validation

A <i>Transaction Payload</i> has different validation stages, since some validation steps can only be executed at the
point when certain information has (or has not) been received. We therefore distinguish between syntactic- and semantic
validation.

The different output types and optional output feature blocks add extra constraints to transaction validation rules,
but since these are specific to the given outputs and features, they are discussed for each
[output types](#output-design) and
[feature blocks types](#optional-output-features).

#### Syntactic Validation

This validation can commence as soon as the transaction data has been received in its entirety. It validates the
structure of the transaction. If the transaction does not pass this stage, it must not be broadcast further and can be
discarded right away.

The following criteria defines whether the transaction passes the syntactic validation:
* `Transaction Essence Type` value must be 0, denoting an `Transaction Essence`.
* Inputs:
    * `Inputs Count` must be 0 < x ≤ `Max Inputs Count`.
    * At least one input must be specified.
    * `Input Type` value must be 0, denoting an `UTXO Input`.
    * `UTXO Input`:
        * `Transaction Output Index` must be 0 ≤ x < `Max Outputs Count`.
        * Every combination of `Transaction ID` + `Transaction Output Index` must be unique in the list of inputs.
* Outputs:
    * `Outputs Count` must be 0 < x ≤ `Max Outputs Count`.
    * At least one output must be specified.
    * `Output Type` must denote a `SimpleOutput`, `ExtendedOutput`, `AliasOutput`, `FoundryOutput` or `NFTOutput`.
    * Output must fulfill the [dust protection requirements.](TODO)
    * Output is syntactically valid based on its type.
    * Accumulated output balance must not exceed the total supply of tokens `2'779'530'283'277'761`.
* `Payload Length` must be 0 (to indicate that there's no payload) or be valid for the specified payload type.
* `Payload Type` must be one of the supported payload types if `Payload Length` is not 0.
* `Unlock Blocks Count` must match the amount of inputs. Must be 0 < x ≤ `Max Inputs Count`.
* `Unlock Block Type` must either be 0, 1, 2 or 3, denoting a `Signature Unlock Block`, a `Reference Unlock block`, an
  `Alias Unlock Block` or an `NFT Unlock Block`.
* `Signature Unlock Blocks` must define a `Ed25519 Signature`.
* A `Signature Unlock Block` unlocking multiple inputs must only appear once (be unique) and be positioned at the same
  index of the first input it unlocks. All other inputs unlocked by the same `Signature Unlock Block` must have a
  companion `Reference Unlock Block` at the same index as the corresponding input which points to the origin
  `Signature Unlock Block`.
* `Reference Unlock Blocks` must specify a previous `Unlock Block` which is not of type `Reference Unlock Block`. The
  referenced index must therefore be < the index of the `Reference Unlock Block`.
* `Alias Unlock Blocks` must specify a previous `Unlock Block` which unlocks the alias the input is locked to. The
  referenced index must be < the index of the `Alias Unlock Block`.
* `NFT Unlock Blocks` must specify a previous `Unlock Block` which unlocks the NFT the input is locked to. The
  reference index must be < the index of the `NFT Unlock Block`.
* Given the type and length of the information, the <i>Transaction Payload</i> must consume the entire byte array for
  the `Payload Length` field in the <i>Message</i> it defines.

#### Semantic Validation

Semantic validation starts when a message that is part of a confirmation cone of a milestone is processed in the
[White-Flag ordering](https://github.com/iotaledger/protocol-rfcs/blob/master/text/0005-white-flag/0005-white-flag.md#deterministically-ordering-the-tangle).
Semantics are only validated during White-Flag confirmations to enforce an ordering that can be understood by all the
nodes (i.e. milestone cones), no matter the order the transactions are received. Otherwise, if semantic validation
would occur as soon as a transaction would be received, it could potentially lead to nodes having different views of
the UTXOs available to spend.


Processing transactions in the White-Flag ordering enables users to spend UTXOs which are in the same milestone
confirmation cone, if their transaction comes after the funding transaction in the mentioned White-Flag ordering. It is
recommended that users spending unconfirmed UTXOs attach their message directly onto the message containing the source
transaction.

The following criteria defines whether the transaction passes the semantic validation:
1. The UTXOs the transaction references must be known (booked) and unspent.
2. The transaction is spending the entirety of the IOTA coins of the referenced UTXOs to the outputs.
3. The transaction is balanced in terms of native tokens, meaning the amount of native tokens present in inputs equals
   to that of outputs. Otherwise, the foundry outputs controlling outstanding native token balances must be present in
   the transaction. The validation of the foundry output(s) determines if the outstanding balances are valid.
4. The UTXOs the transaction references must be unlocked based on the transaction context, that is the
   <i>Transaction Payload</i> plus the list of consumed UTXOs. (output syntactic unlock validation in transaction
   context)
5. The UTXOs the transaction references must be unlocked with respect to the
   [milestone index and Unix timestamp of the confirming milestone](https://github.com/jakubcech/protocol-rfcs/blob/jakubcech-milestonepayload/text/0019-milestone-payload/0019-milestone-payload.md#structure). (output semantic unlock validation in transaction context)
6. The outputs of the transaction must pass additional validation rules defined by the present
   [output feature blocks](#optional-output-features).
7. The sum of all `Native Token Counts` in the transaction plus `Outputs Count` is ≤
   `Max Native Token Count Per Output`.
8. The address type of the referenced UTXO (input) must match that of the corresponding <i>Unlock Block</i>.
9. The <i>Signature Unlock Blocks</i> are valid, i.e. the signatures prove ownership over the addresses of the
   referenced UTXOs.


If a transaction passes the semantic validation, its referenced UTXOs must be marked as spent and the corresponding new
outputs must be booked/specified in the ledger. The booked transaction then also becomes part of the White-Flag Merkle
tree inclusion set.

Transactions which do not pass semantic validation are ignored. Their UTXOs are not marked as spent and neither are
their outputs booked into the ledger.

### Summary of Changes

- Deprecating <i>SigLockedSingleOutput</i> and <i>SigLockedDustAllowanceOutput</i>.
    - The new dust protection mechanism does not need a distinct output type, therefore
      <i>SigLockedDustAllowanceOutput</i> will be deprecated. One alternative is that during migration to the new
      protocol version, all dust outputs sitting on an address will be merged into a `SimpleOutput` together with their
      respective <i>SigLockedDustAllowanceOutputs</i> to create the snapshot for the updated protocol. The exact
      migration strategy will be decided later.
- Introducing new output types [discussed later](#output-design).
- <i>Inputs</i> and <i>Outputs</i> of a transaction become a list instead of a set. Binary duplicate inputs are not
  allowed as they anyway mean double-spends, but binary duplicate outputs are allowed.
- There can be many outputs created to the same address in the transaction.
- Confirming milestone supplies notion of time to semantic transaction validation.

## New Concepts

New output types add new features to the protocol and hence new transaction validation rules. While some of these new
features are specifically tied to one output type, some are general, LEGO like building blocks that may be put in
several types of outputs. Below is a summary of such  new features and the validation rules they introduce.

### Native Tokens in Outputs

Outputs are records in the UTXO ledger that track ownership of funds. Thus, each output must be able to specify which
funds it holds. With the addition of the Native Tokenization Framework, outputs may also carry user defined native
tokens, that is, tokens that are not IOTA coins but were minted by foundries and are tracked in the very same ledger.
Therefore, **every output must be able to hold not only IOTA coins, but also native tokens**.

The only exception to this rule is the <i>SimpleOutput</i> (formerly known as <i>SigLockedSingleOutput</i>), which is
kept for backwards compatibility.

Dust protection applies to all outputs, therefore it is not possible for outputs to hold only native tokens, the dust
requirements must be covered via IOTA coins.

User defined tokens are called <i>Native Tokens</i> on protocol level. The maximum supply of a particular native token
is defined by the representation chosen on protocol level for defining their amounts in outputs. Since native tokens
are also a vehicle to wrap layer 2 tokens into layer 1 tokens, the chosen representation must take into account the
maximum possible supply of layer 2 tokens. Solidity, the most popular layer 2 smart contract language defines the
maximum supply of an ERC-20 token as `MaxUint256`, therefore it should be possible represent such huge amount of assets
on layer 1.

Outputs must have the following fields to define the balance of native tokens they hold:

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Native Tokens Count</td>
        <td>uint16</td>
        <td>The number of native tokens present in the output.</td>
    </tr>
    <tr>
        <td valign="top">Native Tokens <code>optAnyOf</code></td>
        <td colspan="2">
            <details>
                <summary>Native Token</summary>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Token ID</td>
                        <td>ByteArray[38]</td>
                        <td>
                            Identifier of the native token. Derivation defined <a href=https://github.com/lzpap/protocol-rfcs/blob/master/text/0038-output-types-for-sc-and-tokenization/0038-output-types-for-sc-and-tokenization.md#foundry-output>here</a>.
                        </td>
                    </tr>
                    <tr>
                        <td>Amount</td>
                        <td>uint256</td>
                        <td>
                            Amount of tokens.
                        </td>
                    </tr>
                </table>
            </details>
        </td>
    </tr>
</table>

#### Additional semantic transaction validation rules:

- The transaction is balanced in terms of native tokens, that is, the sum of native token balances in consumed outputs
  equals that of the created outputs.
- When the transaction is imbalanced, the foundry outputs controlling outstanding native token balances must be present
  in the transaction. The validation of the foundry output(s) determines if the outstanding balances are valid.


### Optional Output Features

The programmability of outputs opens the door for implementing new features for the base protocol. While some outputs
were specifically designed for such new features, some are optional additions that may be used with any outputs that
support them. These optional features are called **Feature Blocks**, building blocks that can be put into outputs that
support them.

Each output **must not contain more than one feature block of each type** and not all block types are supported for
each output type.

Below is a summary of all available and optional feature blocks.

#### Sender Block

Every transaction consumes several elements from the UTXO set and creates new outputs. However, certain applications
(smart contracts) require to associate each output with exactly one sender address. Here, the sender block is used to
specify the validated sender of an output.

Outputs that support the <i>Sender Block</i> may specify a `Sender` address which is validated by the protocol during
transaction validation.

##### Additional semantic transaction validation rule:
- The <i>Sender Block</i>, and hence the output and transaction that contain it, is valid, if and only if an output
- with the corresponding address is consumed and unlocked in the transaction.

<details>
<summary>Sender Block</summary>
<blockquote>
        Identifies the validated sender of the output.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 0</strong> to denote a <i>Sender Block</i>.
        </td>
    </tr>
    <tr>
        <td valign="top">Sender <code>oneOf</code></td>
        <td colspan="2">
            <details>
                <summary>Ed25519 Address</summary>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Address Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Address</td>
                        <td>ByteArray[32]</td>
                        <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                    </tr>
                </table>
            </details>
            <details>
                <summary>Alias Address</summary>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Address Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Address</td>
                        <td>ByteArray[20]</td>
                        <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                    </tr>
                </table>
            </details>
            <details>
                <summary>NFT Address</summary>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Address Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 16</strong> to denote an <i>NFT Address</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Address</td>
                        <td>ByteArray[20]</td>
                        <td>The raw bytes of the nft address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                    </tr>
                </table>
            </details>
        </td>
    </tr>
</table>

:::info
The <i>Address Type</i> byte of a raw address has an effect on the starting character of the bech32 encoded address,
which is the recommended address format for user facing applications.

A usual bech32 encoded mainnet address starts with `iota1`, and continues with the bech32 encoded bytes of the address.
By choosing <i>Address Type</i> as a multiple of 8 for different address types, the first character after the `1`
separator in the bech32 address will always be different.

| Address | Type Byte | Bech32 Encoded |
| ------- | --------- | -------------- |
| Ed25519 | 0 | iota1**q**... |
| Alias | 8 | iota1**p**... |
| NFT | 16 | iota1**z**... |

A user can identify by looking at the address wether it is a signature backed address, a smart contract chain account
or an NFT address.
:::

#### Issuer Block

The issuer block is a special case of the sender block that it is only supported by outputs that implement a UTXO state
machine with [chain constraint](#chain-constraint-in-utxo) (alias, NFT).
Only when the state machine is created (e.g. minted) it is checked during transaction validation that an output
corresponding to the `Issuer` address is consumed. In every future transition of the state machine, it is instead
checked that the issuer block is still present and unchanged.

##### Additional semantic transaction validation rule:
- When an <i>Issuer Block</i> is present in an output representing the initial state of an UTXO state machine, the
- transaction that contains this output is valid, if and only if an output with the corresponding address is consumed
- and unlocked in the transaction.

The main use case is proving authenticity of NFTs. Whenever an NFT is minted as an NFT output, the creator (issuer) can
fill the `Issuer Block` with their address that they have to unlock in the transaction. Issuers then can publicly
disclose their addresses to prove the authenticity of the NFT once it is in circulation.

<details>
<summary>Issuer Block</summary>
<blockquote>
        Identifies the validated issuer of the output.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 1</strong> to denote an <i>Issuer Block</i>.
        </td>
    </tr>
    <tr>
        <td valign="top">Issuer <code>oneOf</code></td>
        <td colspan="2">
            <details>
                <summary>Ed25519 Address</summary>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Address Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Address</td>
                        <td>ByteArray[32]</td>
                        <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                    </tr>
                </table>
            </details>
            <details>
                <summary>Alias Address</summary>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Address Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Address</td>
                        <td>ByteArray[20]</td>
                        <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                    </tr>
                </table>
            </details>
            <details>
                <summary>NFT Address</summary>
                <table>
                    <tr>
                        <td><b>Name</b></td>
                        <td><b>Type</b></td>
                        <td><b>Description</b></td>
                    </tr>
                    <tr>
                        <td>Address Type</td>
                        <td>uint8</td>
                        <td>
                            Set to <strong>value 16</strong> to denote an <i>NFT Address</i>.
                        </td>
                    </tr>
                    <tr>
                        <td>Address</td>
                        <td>ByteArray[20]</td>
                        <td>The raw bytes of the nft address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                    </tr>
                </table>
            </details>
        </td>
    </tr>
</table>

Whenever a chain account mints an NFT on layer 1 on behalf of some user, the `Issuer` field can only contain the
chain's address, since user does not sign the layer 1 transaction. As a consequence, artist would have to mint NFTs
themselves on layer 1 and then deposit it to chains if they want to place their own address in the `Issuer` field.

#### Return Amount Block

This block is always used in combination with the <i>Sender Block</i> to achieve conditional sending. An output that
has both `Sender` and `Return Amount` specified can only be consumed in a transaction that deposits `Return Amount`
IOTA coins into `Sender` address. When several of such outputs are consumed, their return amounts per `Sender`
addresses are summed up and the output side must deposit this total sum per `Sender` address.

##### Additional syntactic transaction validation rule:

- An output that has <i>Return Amount Block</i> specified must have the <i>Sender Block</i> specified too.
- `Return Amount` must be ≥ than `Minimum Dust Deposit`.
- `Return Amount` in a <i>Return Amount Block</i> must be ≤ than the required dust deposit of the output.

##### Additional semantic transaction validation rule:

- An output that has <i>Return Amount Block</i> specified must only be consumed and unlocked in a transaction that
  deposits `Return Amount` IOTA coins to `Sender` address via an output that has no additional spending constraints.
  (<i>SimpleOutput</i> or <i>ExtendedOutput</i> without feature blocks)
- When several outputs with <i>Return Amount Block</i> and the same `Sender` are consumed, their return amounts per
  `Sender` addresses are summed up and the output side of the transaction must deposit this total sum per `Sender`
  address.

<details>
<summary>Return Amount Block</summary>
<blockquote>
        Defines the amount of IOTAs that have to be returned to <i>Sender</i>.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 2</strong> to denote a <i>Return Amount Block</i>.
        </td>
    </tr>
    <tr>
        <td>Return Amount</td>
        <td>uint64</td>
        <td>
            Amount of IOTA coins the consuming transaction should deposit to the address defined in <i>Sender Block</i>.
        </td>
    </tr>
</table>

This feature block makes it possible to send small amounts of IOTA coins or native tokens to addresses without having
to lose control of the required dust deposit. It is also a vehicle to send on-chain requests to ISCP chains that do not
require fees. To prevent the receiving party from blocking access to the dust deposit, it is advised to be used
together with the [Expiration Blocks](#Expiration-Blocks). The receiving party then has a sender-defined time window to
agree to the transfer by consuming the output, or the sender regains total control after expiration.

#### Timelock Blocks

The notion of time in the Tangle is introduced via milestones. Each milestone
[carries the current milestone index and the unix timestamp](https://github.com/jakubcech/protocol-rfcs/blob/jakubcech-milestonepayload/text/0019-milestone-payload/0019-milestone-payload.md#structure) corresponding to that index. Whenever a new milestone appears, nodes perform the white-flag ordering and transaction validation on its past cone. The timestamp and milestone index of the confirming milestone provide the time as an input parameter to transaction validation.

An output that contains a <i>Timelock Block</i> can not be unlocked before the specified timelock has expired. The
timelock is expired when the timestamp and/or milestone index of the confirming milestone is past the timestamp and/or
milestone defined in the <i>Timelock Block</i>.

The timelock can be specified as a unix timestamp or as a milestone index. When specified in both ways, both conditions
have to pass in order for the unlock to be valid.

The two time representations help to protect against the possible downtime of the Coordinator. If the Coordinator is
down, "milestone index clock" essentially stops advancing, while "real time clock" does not. An output that specifies
time in both clocks must satisfy both conditions (AND relation).

##### Additional semantic transaction validation rules:
- An output that has <i>Timelock Milestone Index Block</i> specified must only be consumed and unlocked in a
  transaction, if the confirming milestone index is > than the `Milestone Index` specified in the block.
- An output that has <i>Timelock Unix Block</i> specified must only be consumed and unlocked in a transaction, if the
  timestamp of the confirming milestone is past the `Unix Time` specified in the block.
- An output that has both <i>Timelock Milestone Index Block</i> and  <i>Timelock Unix Block</i> specified must only be
  consumed and unlocked in a transaction if both blocks validate.

<details>
<summary>Timelock Milestone Index Block</summary>
<blockquote>
    Defines a milestone index until which the output can not be unlocked.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 3</strong> to denote a <i>Timelock Milestone Index Block</i>.
        </td>
    </tr>
    <tr>
        <td>Milestone Index</td>
        <td>uint32</td>
        <td>
            The milestone index starting from which the output can be consumed.
        </td>
    </tr>
</table>

<details>
<summary>Timelock Unix Block</summary>
<blockquote>
    Defines a unix time until which the output can not be unlocked.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 4</strong> to denote a <i>Timelock Unix Block</i>.
        </td>
    </tr>
    <tr>
        <td>Unix Time</td>
        <td>uint32</td>
        <td>
            Unix time (seconds since Unix epoch) starting from which the output can be consumed.
        </td>
    </tr>
</table>

#### Expiration Blocks

The expiration feature of outputs makes it possible for the sender to reclaim an output after a given expiration time
has been passed. As a consequence, expiration blocks may only be used in the presence of the <i>Sender Block</i>. The
expiration might be specified as a unix timestamp or as a milestone index. When specified in both ways, both conditions
have to pass in order for the unlock to be valid.

The expiration feature can be viewed as an opt-in receive feature, because the recipient loses access to the received
funds after the output expires, while the sender regains control over them. This feature is a big help for on-chain
smart contract requests. Those that have expiration set and are sent to dormant smart contract chains can be recovered
by their senders. Not to mention the possibility to time requests by specifying both a timelock and an expiration block.

##### Additional syntactic transaction validation rules:

- An output that has either <i>Expiration Milestone Index Block</i> or <i>Expiration Unix Block</i> specified must have
  the <i>Sender Block</i> specified.

##### Additional semantic transaction validation rules:

- An output that has <i>Expiration Milestone Index Block</i> set must only be consumed and unlocked by the target
  `Address` in a transaction that has a confirming milestone index < than the `Milestone Index` defined in the block.
- An output that has <i>Expiration Milestone Index Block</i> set must only be consumed and unlocked by the `Sender`
  address in a transaction that has a confirming milestone index ≥ than the `Milestone Index` defined in the block.
- An output that has <i>Expiration Unix Block</i> set must only be consumed and unlocked by the target `Address` in a
  transaction that has a confirming milestone timestamp earlier than the `Unix Time` defined in the block.
- An output that has <i>Expiration Unix Block</i> set must only be consumed and unlocked by the `Sender` address in a
  transaction that has a confirming milestone timestamp same or later than the `Unix Time` defined in the block.

<details>
<summary>Expiration Milestone Index Block</summary>
<blockquote>
    Defines a milestone index until which only <i>Address</i> is allowed to unlock the output. After the milestone index, only <i>Sender</i> can unlock it.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 5</strong> to denote a <i>Expiration Milestone Index Block</i>.
        </td>
    </tr>
    <tr>
        <td>Milestone Index</td>
        <td>uint32</td>
        <td>
            Before this milestone index, <i>Address</i> is allowed to unlock the output, after that only the address defined in <i>Sender Block</i>.
        </td>
    </tr>
</table>

<details>
<summary>Expiration Unix Block</summary>
<blockquote>
    Defines a unix time until which only <i>Address</i> is allowed to unlock the output. After the expiration time, only <i>Sender</i> can unlock it.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 6</strong> to denote a <i>Expiration Unix Block</i>.
        </td>
    </tr>
    <tr>
        <td>Unix Time</td>
        <td>uint32</td>
        <td>
            Before this unix time (seconds since Unix epoch), <i>Address</i> is allowed to unlock the output, after that only the address defined in <i>Sender Block</i>.
        </td>
    </tr>
</table>

#### Metadata Block

Outputs may carry additional data with them that is interpreted by higher layer applications built on the Tangle. The
protocol treats this metadata as pure binary data, it has no effect on the validity of an output except that it
increases the required dust deposit. ISCP is a great example of a higher layer protocol that makes use of
<i>Metadata Block</i>: smart contract request parameters are encoded in the metadata field of outputs.

##### Additional syntactic transaction validation rules:
- An output with <i>Metadata Block</i> is valid, if and only if 0 < `Data Length` ≤ `Max Metadata Length`.

<details>
<summary>Metadata Block</summary>
<blockquote>
    Defines metadata (arbitrary binary data) that will be stored in the output.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 7</strong> to denote a <i>Metadata Block</i>.
        </td>
    </tr>
    <tr>
        <td>Data Length</td>
        <td>uint32</td>
        <td>
            Length of the following data field in bytes.
        </td>
    </tr>
    <tr>
        <td>Data</td>
        <td>ByteArray</td>
        <td>Binary data.</td>
    </tr>
</table>

#### Indexation Block

An indexation block makes it possible to tag outputs with an index, so they can be retrieved through an API not only by
their address, but also based on the the `Indexation Tag`. **The combination of an <i>Indexation Block</i>, a
<i>Metadata Block</i> and a <i>Sender Block</i> makes it possible to retrieve data associated to an address and stored
in outputs that was created by a specific party (`Sender`) for a specific purpose (`Indexation Tag`).**

Storing indexed data in outputs however incurs greater dust deposit for such outputs, because they create look-up
entries in nodes' databases.

##### Additional syntactic transaction validation rules:
- An output with <i>Indexation Block</i> is valid, if and only if 0 < `Indexation Tag Length` ≤
  `Max Indexation Tag Length`.

<details>
<summary>Indexation Block</summary>
<blockquote>
    Defines an indexation tag to which the <i>Metadata Block</i> will be indexed. Creates an indexation lookup in nodes.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 8</strong> to denote a <i>Indexation Block</i>.
        </td>
    </tr>
    <tr>
        <td>Indexation Tag Length</td>
        <td>uint32</td>
        <td>
            Length of the following indexation tag field in bytes.
        </td>
    </tr>
    <tr>
        <td>Indexation Tag</td>
        <td>ByteArray</td>
        <td>Binary indexation tag.</td>
    </tr>
</table>



### Chain Constraint in UTXO

Previously created transaction outputs are destroyed when they are consumed in a subsequent transaction as an input.
The chain constraint makes it possible to **carry the UTXO state machine state encoded in outputs across transactions.**
When an output with chain constraint is consumed, that transaction has to create a single subsequent output that
carries the state forward. The **state can be updated according to the transition rules defined for the given type of
output and its current state**. As a consequence, each such output has a unique successor, and together they form a path
or *chain* in the graph induced by the UTXO spends. Each chain is identified by its globally unique identifier.

![](https://i.imgur.com/izQ4DB1.png)

Alias outputs, foundry outputs and NFT outputs all use this chain constraint concept and define their own unique
identifiers.

## Output Design

In the following, we define four new output types. They are all designed with specific use cases in mind:
- **Extended Output**: transfer of funds with attached metadata and optional spending restrictions. Main use cases are
  on-ledger ISCP requests, native asset transfers and indexed data storage in the UTXO ledger.
- **Alias Output**: representing ISCP chain accounts on L1 that can process requests and transfer funds.
- **Foundry Output**: supply control of user defined native tokens. A vehicle for cross-chain asset transfers and asset
  wrapping.
- **NFT Output**: an output that represents a Non-fungible token with attached metadata and proof-of-origin. A NFT is
  represented as an output so that the token and metadata are transferred together, for example as a smart contract
  requests. NFTs are possible to implement with native tokens as well, but then ownership of the token does not mean
  ownership of the foundry that holds its metadata.

The validation of outputs is part of the transaction validation process. There are two levels of validation for
transactions: syntactic and semantic validation. The former validates the structure of the transaction (and outputs),
while the latter validates whether protocol rules are respected in the semantic context of the transaction. Outputs
hence are validated on both levels:
1. **Transaction Syntactic Validation**: validates the structure of each output created by the transaction.
2. **Transaction Semantic Validation**:
    - **For consumed outputs**: validates whether the output can be unlocked in a transaction given the semantic
      transaction context.
    - **For created outputs**: validates whether the output can be created in a transaction given the semantic
      transaction context.

Each new output type may add its own validation rules which become part of the transaction validation rules if the
output is placed inside a transaction. Optional feature blocks described previously also add constraints to transaction
validation when they are placed in outputs.

## SimpleOutput

The <i>SigLockedSingleOutput</i> introduced in Chrysalis Pt2 is kept for backwards compatibility but renamed to
<i>SimpleOutput</i>, although its simple feature, holding IOTA coins is supported by an <i>Extended Output</i>.

Every <i>SigLockedSingleOutput</i> is a valid <i>SimpleOutput</i>, but not the other way around.
<i>SigLockedSingleOutput</i> only supports <i>Ed25519 Address</i> in the `Address` field, while <i>SimpleOutput</i>
also supports <i>Alias Address</i> and <i>NFT Address</i>.

Next to the use case of holding only IOTA coins, simple outputs are also a vehicle for conditional sending
(<i>Return Amount Block</i>). A return amount specified within such a block must be sent back to `Sender` via a
<i>Simple Output</i> or an <i>Extended Output</i> without optional feature blocks, since these have the lowest minimum
dust requirements among all outputs and no additional spending conditions can be defined on them.

<table>
    <details>
        <summary>Simple Output</summary>
        <blockquote>
            Describes a simple output that can only hold IOTAs. For backwards compatibility reasons, this is the same as a SigLockedSingleOutput.
        </blockquote>
        <table>
            <tr>
                <td><b>Name</b></td>
                <td><b>Type</b></td>
                <td><b>Description</b></td>
            </tr>
            <tr>
                <td>Output Type</td>
                <td>uint8</td>
                <td>
                    Set to <strong>value 0</strong> to denote an <i>Simple Output</i>.
                </td>
            </tr>
            <tr>
                <td valign="top">Address <code>oneOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Ed25519 Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[32]</td>
                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Alias Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>NFT Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 16</strong> to denote an <i>NFT Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the nft address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Amount</td>
                <td>uint64</td>
                <td>The amount of IOTA coins to held by the output.</td>
            </tr>
        </table>
    </details>
</table>

## Extended Output

<i>ExtendedOutput</i> is basically a <i>SimpleOutput</i> that can hold native tokens and might have optional feature
blocks. The combination of several feature blocks provide the base functionality for the output to be used as an
on-ledger smart contract request:
- Verified `Sender`,
- Attached `Metadata` that can encode the request payload for layer 2,
- `Return Amount` to get back the dust deposit,
- `Timelock` to be able to time requests,
- `Expiration` to recover funds in case of chain inactivity.

Besides, the <i>Indexation Block</i> feature is a tool to store arbitrary, indexed data with verified origin in the
ledger.

<table>
    <details>
        <summary>Extended Output</summary>
        <blockquote>
            Describes an extended output with optional features.
        </blockquote>
        <table>
            <tr>
                <td><b>Name</b></td>
                <td><b>Type</b></td>
                <td><b>Description</b></td>
            </tr>
            <tr>
                <td>Output Type</td>
                <td>uint8</td>
                <td>
                    Set to <strong>value 1</strong> to denote an <i>Extended Output</i>.
                </td>
            </tr>
            <tr>
                <td valign="top">Address <code>oneOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Ed25519 Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[32]</td>
                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Alias Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>NFT Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 16</strong> to denote an <i>NFT Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the nft address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Amount</td>
                <td>uint64</td>
                <td>The amount of IOTA coins to held by the output.</td>
            </tr>
            <tr>
                <td>Native Token Count</td>
                <td>uint16</td>
                <td>The number of native tokens held by the output.</td>
            </tr>
            <tr>
                <td valign="top">Native Tokens <code>optAnyOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Native Token</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Token ID</td>
                                <td>ByteArray[38]</td>
                                <td>
                                    Identifier of the native token.
                                </td>
                            </tr>
                            <tr>
                                <td>Amount</td>
                                <td>uint256</td>
                                <td>
                                    Amount of native tokens of the given <i>Token ID</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Blocks Count</td>
                <td>uint16</td>
                <td>The number of feature blocks following.</td>
            </tr>
            <tr>
                <td valign="top">Blocks <code>atMostOneOfEach</code></td>
                <td colspan="2">
                    <details>
                        <summary>Sender Block</summary>
                        <blockquote>
                            Identifies the validated sender of the output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote a <i>Sender Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td valign="top">Sender <code>oneOf</code></td>
                                <td colspan="2">
                                    <details>
                                        <summary>Ed25519 Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[32]</td>
                                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>BLS Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 1</strong> to denote a <i>BLS Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[32]</td>
                                                <td>The raw bytes of the BLS address which is a BLAKE2b-256 hash of the BLS public key.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>Alias Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[20]</td>
                                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                                            </tr>
                                        </table>
                                    </details>
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Return Amount Block</summary>
                        <blockquote>
                            Defines the amount of IOTAs that have to be returned to <i>Sender</i>.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 2</strong> to denote a <i>Return Amount Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Return Amount</td>
                                <td>uint64</td>
                                <td>
                                    Amount of IOTA tokens the consuming transaction should deposit to the address defined in <i>Sender Block</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Timelock Milestone Index Block</summary>
                        <blockquote>
                            Defines a milestone index until which the output can not be unlocked.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 3</strong> to denote a <i>Timelock Milestone Index Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Milestone Index</td>
                                <td>uint32</td>
                                <td>
                                    The milestone index starting from which the output can be consumed.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Timelock Unix Block</summary>
                        <blockquote>
                            Defines a unix time until which the output can not be unlocked.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 4</strong> to denote a <i>Timelock Unix Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Unix Time</td>
                                <td>uint32</td>
                                <td>
                                    Unix time (seconds since Unix epoch) starting from which the output can be consumed.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Expiration Milestone Index Block</summary>
                        <blockquote>
                            Defines a milestone index until which only <i>Address</i> is allowed to unlock the output. After the milestone index, only <i>Sender</i> can unlock it.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 5</strong> to denote a <i>Expiration Milestone Index Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Milestone Index</td>
                                <td>uint32</td>
                                <td>
                                    Before this milestone index, <i>Address</i> is allowed to unlock the output, after that only the address defined in <i>Sender Block</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Expiration Unix Block</summary>
                        <blockquote>
                            Defines a unix time until which only <i>Address</i> is allowed to unlock the output. After the expiration time, only <i>Sender</i> can unlock it.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 6</strong> to denote a <i>Expiration Unix Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Unix Time</td>
                                <td>uint32</td>
                                <td>
                                    Before this unix time, <i>Address</i> is allowed to unlock the output, after that only the address defined in <i>Sender Block</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Metadata Block</summary>
                        <blockquote>
                            Defines metadata (arbitrary binary data) that will be stored in the output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 7</strong> to denote a <i>Metadata Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Data Length</td>
                                <td>uint32</td>
                                <td>
                                    Length of the following data field in bytes.
                                </td>
                            </tr>
                            <tr>
                                <td>Data</td>
                                <td>ByteArray</td>
                                <td>Binary data.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Indexation Block</summary>
                        <blockquote>
                            Defines an indexation tag to which the <i>Metadata Block</i> will be indexed. Creates an indexation lookup in nodes.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote a <i>Indexation Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Indexation Tag Length</td>
                                <td>uint32</td>
                                <td>
                                    Length of the following indexation tag field in bytes.
                                </td>
                            </tr>
                            <tr>
                                <td>Indexation Tag</td>
                                <td>ByteArray</td>
                                <td>Binary indexation data.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
        </table>
    </details>
</table>

### Additional Transaction Syntactic Validation Rules

- `Amount` field must fulfill the dust protection requirements.
- `Native Token Count` must not be greater than `Max Native Token Count Per Output`.
- It must hold true that `0` ≤ `Blocks Count` ≤ `8`.
- <i>Blocks</i> must be sorted in ascending order based on their `Block Type`.
- Syntactic validation of all present feature blocks must pass.

### Additional Transaction Semantic Validation Rules

#### Consumed Outputs

- The unlock block of the input must correspond to `Address` field and the unlock must be valid.
- The unlock is valid if and only if all feature blocks present in the output validate.

#### Created Outputs

- Feature block imposed transaction validation criteria must be fulfilled.

## Alias Output

The <i>Alias Output</i> is a specific implementation of a UTXO state machine. `Alias ID`, the unique identifier of an
instance of the deployed state machine, is generated deterministically by the protocol and is not allowed to change in
any future state transitions.

<i>Alias Output</i> represents an alias account in the ledger with two control levels and a permanent
<i>Alias Address</i>. The account owns other outputs that are locked under <i>Alias Address</i>. The account keeps
track of state transitions (`State Index` counter), controlled foundries (`Foundry Counter`) and anchors the layer 2
state as metadata into the UTXO ledger.

<table>
    <details>
        <summary>Alias Output</summary>
        <blockquote>
            Describes an alias account in the ledger that can be controlled by the state and governance controllers.
        </blockquote>
        <table>
            <tr>
                <td><b>Name</b></td>
                <td><b>Type</b></td>
                <td><b>Description</b></td>
            </tr>
            <tr>
                <td>Output Type</td>
                <td>uint8</td>
                <td>
                    Set to <strong>value 2</strong> to denote a <i>Alias Output</i>.
                </td>
            </tr>
            <tr>
                <td>Amount</td>
                <td>uint64</td>
                <td>The amount of IOTA tokens held by the output.</td>
            </tr>
                        <tr>
                <td>Native Token Count</td>
                <td>uint16</td>
                <td>The number of native tokens held by the output.</td>
            </tr>
            <tr>
                <td valign="top">Native Tokens <code>optAnyOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Native Token</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Token ID</td>
                                <td>ByteArray[38]</td>
                                <td>
                                    Identifier of the native token.
                                </td>
                            </tr>
                            <tr>
                                <td>Amount</td>
                                <td>uint256</td>
                                <td>
                                    Amount of native tokens of the given <i>Token ID</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Alias ID</td>
                <td>ByteArray[20]</td>
                <td>Unique identifier of the alias, which is the BLAKE2b-160 hash of the <i>Output ID</i> that created it.<i> Alias Address = Alias Address Type || Alias ID</i></td>
            </tr>
            <tr>
                <td valign="top">State Controller <code>oneOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Ed25519 Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[32]</td>
                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Alias Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td valign="top">Governance Controller <code>oneOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Ed25519 Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[32]</td>
                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Alias Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>State Index</td>
                <td>uint32</td>
                <td>A counter that must increase by 1 every time the alias is state transitioned.</td>
            </tr>
            <tr>
                <td>State Metadata Length</td>
                <td>uint32</td>
                <td>Length of the following State Metadata field.</td>
            </tr>
            <tr>
                <td>State Metadata</td>
                <td>ByteArray</td>
                <td>Metadata that can only be changed by the state controller.</td>
            </tr>
            <tr>
                <td>Foundry Counter</td>
                <td>uint32</td>
                <td>A counter that denotes the number of foundries created by this alias account.</td>
            </tr>
            <tr>
                <td>Blocks Count</td>
                <td>uint16</td>
                <td>The number of feature blocks following.</td>
            </tr>
            <tr>
                <td valign="top">Blocks <code>atMostOneOfEach</code></td>
                <td colspan="2">
                    <details>
                        <summary>Issuer Block</summary>
                        <blockquote>
                            Identifies the validated issuer of the alias output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 1</strong> to denote an <i>Issuer Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td valign="top">Issuer <code>oneOf</code></td>
                                <td colspan="2">
                                    <details>
                                        <summary>Ed25519 Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[32]</td>
                                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>BLS Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 1</strong> to denote a <i>BLS Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[32]</td>
                                                <td>The raw bytes of the BLS address which is a BLAKE2b-256 hash of the BLS public key.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>Alias Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[20]</td>
                                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>NFT Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 16</strong> to denote an <i>NFT Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[20]</td>
                                                <td>The raw bytes of the nft address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                                            </tr>
                                        </table>
                                    </details>
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Metadata Block</summary>
                        <blockquote>
                            Defines metadata (arbitrary binary data) that will be stored in the output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 7</strong> to denote a <i>Metadata Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Data Length</td>
                                <td>uint32</td>
                                <td>
                                    Length of the following data field in bytes.
                                </td>
                            </tr>
                            <tr>
                                <td>Data</td>
                                <td>ByteArray</td>
                                <td>Binary data.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
        </table>
    </details>
</table>

### Additional Transaction Syntactic Validation Rules

#### Output Syntactic Validation

- `Amount` field must fulfill the dust protection requirements.
- `Native Token Count` must not be greater than `Max Native Token Count Per Output`.
- It must hold true that `0` ≤ `Blocks Count` ≤ `2`.
- <i>Blocks</i> must be sorted in ascending order based on their `Block Type`.
- Syntactic validation of all present feature blocks must pass.
- When `Alias ID` is zeroed out, `State Index` and `Foundry Counter` must be `0`.
- `State Metadata Length` must not be greater than `Max Metadata Length`.
- `State Controller` and `Governance Controller` must be different from the alias address derived from `Alias ID`.

### Additional Transaction Semantic Validation Rules

- Explicit `Alias ID`: `Alias ID` is taken as the value of the `Alias ID` field in the alias output.
- Implicit `Alias ID`: When an alias output is consumed as an input in a transaction and `Alias ID` field is zeroed out
  while `State Index` and `Foundry Counter` are zero, take the BLAKE2b-160 hash of the `Output ID` of the input as
  `Alias ID`.
- The BLAKE2b-160 hash function outputs a 20 byte hash as opposed to the 32 byte hash size of BLAKE2b-256. `Alias ID`
  is sufficiently secure and collision free with 20 bytes already, as it is actually a hash of an `Output ID`, which
  already contains the 32 byte hash `Transaction ID`.
- For every non-zero explicit `Alias ID` on the output side there must be a corresponding alias on the input side. The
  corresponding alias has the explicit or implicit `Alias ID` equal to that of the alias on the output side.

#### Consumed Outputs

Whenever an alias output is consumed in a transaction, it means that the alias is transitioned into its next state. The
**current state** is defined as the **consumed alias output**, while the **next state** is defined as the **alias
output with the same explicit `AliasID` on the output side**. There are two types of transitions: `state transition`
and `governance transition`.
- State transition:
    - A state transition is identified by an incremented `State Index`.
    - The `State Index` must be incremented by 1.
    - The unlock block must correspond to the `State Controller`.
    - State transition can only change the following fields in the next state: `IOTA Amount`, `Native Tokens`,
      `State Index`, `State Metadata Length`, `State Metadata` and `Foundry Counter`.
    - `Foundry Counter` field must increase by the number of foundry outputs created in the transaction that map to
      `Alias ID`. The `Serial Number` fields of the created foundries must be the set of natural numbers that cover the
       open-ended interval between the previous and next values of the `Foundry Counter` field in the alias output.
    - The created foundry outputs must be sorted in the list of outputs by their `Serial Number`. Note, that any
      foundry that maps to `Alias ID` and has a `Serial Number` that is less or equal to the `Foundry Counter` of the
      input alias is ignored when it comes to sorting.
    - Newly created foundries in the transaction that map to different aliases can be interleaved when it comes to
      sorting.
- Governance transition:
    - A governance transition is identified by an unchanged `State Index` in next state. If there is no alias output on
      the output side with a corresponding explicit `Alias ID`, the alias is being destroyed. The next state is the
      empty state.
    - The unlock block must correspond to the `Governance Controller`.
    - Governance transition must only change the following fields: `State Controller`, `Governance Controller`,
      `Metadata Block`.
    - The `Metadata Block` is optional, the governor can put additional info about the chain here, for example chain
      name, fee structure, supported VMs, list of access nodes, etc., anything that helps clients to fetch info (i.e.
      account balances) about the layer 2 network.
- When a consumed alias output has an `Issuer Block` and a corresponding alias output on the output side,
  `Issuer Block` is not allowed to change.

#### Created Outputs

- When <i>Issuer Block</i> is present in an output and explicit `Alias ID` is zeroed out, an input with `Address` field
  that corresponds to `Issuer` must be unlocked in the transaction.

### Notes
- `Governance controller` field is made mandatory for now to help formal verification. When the same entity is defined
  for state and governance controllers, the output is self governed. Later, for compression reasons, it is possible to
  make the governance controller optional and define a self-governed alias as one that does not have the governance
  controller set.
- Nodes shall map the alias address of the output derived with `Alias ID` to the regular <i>address -> output</i>
  mapping table, so that given an <i>Alias Address</i>, its most recent unspent alias output can be retrieved.

### Unresolved Questions

- Is an immutable data field needed for alias output?

## Foundry Output

A foundry output is an output that **controls the supply of user defined native tokens.** It can mint and burn tokens
according to the **policy** defined in the `Token Scheme` field of the output. Foundries can only be created and
controlled by aliases.

**The concatenation of `Address` || `Serial Number` || `Token Scheme Type` fields defines the unique identifier of the
foundry, the `Foundry ID`.**

Upon creation of the foundry, the alias defined in the `Address` field must be unlocked in the same transaction, and
its `Foundry Counter` field must increment. This incremented value defines `Serial Number`, while the `Token Scheme`
can be chosen freely.

`Foundry ID` is not allowed to change after deployment, therefore neither `Address`, nor `Serial Number` or
`Token Scheme` can change during the lifetime of the foundry.

Foundries control the supply of tokens with unique identifiers, so-called `Token IDs`. The **`Token ID` of tokens
controlled by a specific foundry is the concatenation of `Foundry ID` || `Token Tag`.**

<table>
    <details>
        <summary>Foundry Output</summary>
        <blockquote>
            Describes a foundry output that is controlled by an alias.
        </blockquote>
        <table>
            <tr>
                <td><b>Name</b></td>
                <td><b>Type</b></td>
                <td><b>Description</b></td>
            </tr>
            <tr>
                <td>Output Type</td>
                <td>uint8</td>
                <td>
                    Set to <strong>value 3</strong> to denote a <i>Foundry Output</i>.
                </td>
            </tr>
            <tr>
                <td>Address <code>oneOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Alias Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Amount</td>
                <td>uint64</td>
                <td>The amount of IOTA coins to held by the output.</td>
            </tr>
            <tr>
                <td>Native Tokens Count</td>
                <td>uint16</td>
                <td>The number of different native tokens held by the output.</td>
            </tr>
            <tr>
                <td valign="top">Native Tokens <code>optAnyOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Native Token</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>TokenID</td>
                                <td>ByteArray[38]</td>
                                <td>
                                    Identifier of the native tokens.
                                </td>
                            </tr>
                            <tr>
                                <td>Amount <code>oneOf</code></td>
                                <td>uint256</td>
                                <td>Amount of native tokens of the given <i>Token ID</i>.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Serial Number</td>
                <td>uint32</td>
                <td>The serial number of the foundry with respect to the controlling alias.</td>
            </tr>
            <tr>
                <td>Token Tag</td>
                <td>ByteArray[12]</td>
                <td>Data that is always the last 12 bytes of ID of the tokens produced by this foundry.</td>
            </tr>
            <tr>
                <td>Circulating Supply</td>
                <td>uint256</td>
                <td>Circulating supply of tokens controlled by this foundry.</td>
            </tr>
            <tr>
                <td>Maximum Supply</td>
                <td>uint256</td>
                <td>Maximum supply of tokens controlled by this foundry.</td>
            </tr>
            <tr>
                <td valign="top">Token Scheme <code>oneOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Simple Token Scheme</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Token Scheme Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote an <i>Simple Token Scheme</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Blocks Count</td>
                <td>uint16</td>
                <td>The number of feature blocks following.</td>
            </tr>
            <tr>
                <td valign="top">Blocks <code>atMostOneOfEach</code></td>
                <td colspan="2">
                    <details>
                        <summary>Metadata Block</summary>
                        <blockquote>
                            Defines metadata (arbitrary binary data) that will be stored in the output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote a <i>Metadata Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Data Length</td>
                                <td>uint32</td>
                                <td>
                                    Length of the following data field in bytes.
                                </td>
                            </tr>
                            <tr>
                                <td>Data</td>
                                <td>ByteArray</td>
                                <td>Binary data.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
        </table>
    </details>
</table>

### Additional Transaction Syntactic Validation Rules

#### Output Syntactic Validation

- `Amount` field must fulfill the dust protection requirements.
- `Native Tokens Count` must not be greater than `Max Native Token Count Per Output`.
- It must hold true that `0` ≤ `Blocks Count` ≤ `1`.
- Syntactic validation of all present feature blocks must pass.
- `Token Scheme Type` must match one of the supported schemes. Any other value results in invalid output.
- `Circulating Supply` must not be greater than `Maximum Supply`.
- `Maximum Supply` must be larger than zero.

### Additional Transaction Semantic Validation Rules

A foundry is essentially a UTXO state machine. A transaction might either create a new foundry with a unique
`Foundry ID`, transition an already existing foundry or destroy it. The current and next states of the state machine
are encoded in inputs and outputs respectively.

- The **current state of the foundry** with `Foundry ID` `X` in a transaction is defined as the consumed foundry output
  where `Foundry ID` = `X`.
- The **next state of the foundry** with `Foundry ID` `X` in a transaction is defined as the created foundry output
  where `Foundry ID` = `X`.
- `Foundry Diff` is the pair of the **current and next state** of the foundry output in the transaction.

| A transaction that... | Current State | Next State |
| --------------------- | --------------| -----------|
| Creates the foundry | Empty | Output with `Foundry ID` |
| Transitions the foundry | Input with `Foundry ID` | Output with `Foundry ID` |
| Destroys the foundry | Input with `Foundry ID` | Empty |

- The foundry output must be unlocked like any other output type belonging to an <i>Alias Address</i>, by transitioning
  the alias in the very same transaction. See section
  [alias unlocking](#unlocking-chain-script-locked-outputs) for more
  details.
- When the current state of the foundry with `Foundry ID` is empty, it must hold true for `Serial Number` in the next
  state, that:
    - `Foundry Counter(InputAlias) < Serial Number <= Foundry Counter(OutputAlias)`
    - An alias can create several new foundries in one transaction. It was written for the alias output that freshly
      created foundry outputs must be sorted in the list of outputs based on their `Serial Number`. No duplicates are
      allowed.
    - The two previous rules make sure that each foundry output produced by an alias has a unique `Serial Number`,
      hence each `Foundry ID` is unique.
- Native tokens present in a transaction are all native tokens present in inputs and outputs of the transaction. Native
  tokens of a transaction must be a set based on their `Token ID`.
- There must be at most one `Token ID` in the native token set of the transaction that maps to a specific `Foundry ID`.
- Let `Token Diff` denote the difference between native token balances of the input and the output side of the
  transaction of the single `Token ID` that maps to the `Foundry ID`. Minting results in excess of tokens on the output
  side (positive diff), burning results in excess on the input side (negative diff). Now, the following conditions must
  hold for `Token Diff`:
    - `Current State(Circulating Supply) + Token Diff = Next State(Circulating Supply)`.
    - When `Current State` is empty, `Current State(Circulating Supply) = 0`.
    - When `Next State` is empty, `Next State(Circulating Supply) = 0`.
- `Token Scheme Validation` takes `Token Diff` and `Foundry Diff` and validates if the scheme constraints are respected.
  This can include validating `Token Tag` part of the `Token IDs` and the `Token Scheme` fields inside the foundry
  output.
    - `Simple Token Scheme` validates that the `Token Tag` part of the `Token ID` (last 12 bytes) matches the
      `Token Tag` field of the foundry output.
    - Additional token schemes will be defined that make use of the `Foundry Diff` as well, for example validating that
      a certain amount of tokens can only be minted/burned after a certain date.
- When neither `Current State` nor `Next State` is empty:
    -  `Maximum Suppply` field must not change.
    -  `Address` must not change.
    -  `Serial Number` must not change.
    -  `Token Tag` must not change.
    -  `Token Scheme Type` must not change.

### Notes

- A token scheme is a list of hard coded constraints. It is not feasible at the moment to foresee the future
  needs/requirements of hard coded constraints, so it is impossible to design token schemes as any possible combination
  of those constraints. A better design would be to have a list of possible constraints (and their related fields) from
  which the user can choose. The chosen combination should still be encoded as a bitmask inside the `Token ID`.
- For now, only token scheme `0` is supported. Additional token schemes will be designed iteratively while doing the
  testnet implementation.
- The `Foundry ID` of a foundry output should have a global mapping table in nodes, so that given a `Foundry ID`, the
  `Output ID` of the foundry output can be retrieved. `Foundry ID` behaves like an address that can't unlock anything.
  While it is not neccessarly needed for the protocol, it is needed for client side operations (what is the current
  state of the foundry? accessing token metadata in foundry based on `Foundry ID` derived from `Tokend ID`).

## NFT Output

Non-fungible tokens in the ledger are implemented with a special output type, the so-called <i>NFTOutput</i>.

Each NFT output gets assigned a unique identifier `NFT ID` upon creation by the protocol. `NFT ID` is BLAKE2b-160 hash
of the <i>Output ID</i>  that created the NFT. The address of the NFT is the concatenation of `NFT Address Type` ||
`NFT ID`.

The NFT may contain immutable metadata set upon creation, and a verified `Issuer`. The output type supports all
optional feature blocks so that the output can be sent as a request to smart contract chain (alias) accounts.

<table>
    <details>
        <summary>NFT Output</summary>
        <blockquote>
            Describes an NFT output, a globally unique token with metadata attached.
        </blockquote>
        <table>
            <tr>
                <td><b>Name</b></td>
                <td><b>Type</b></td>
                <td><b>Description</b></td>
            </tr>
            <tr>
                <td>Output Type</td>
                <td>uint8</td>
                <td>
                    Set to <strong>value 4</strong> to denote a <i>NFT Output</i>.
                </td>
            </tr>
            <tr>
                <td valign="top">Address <code>oneOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Ed25519 Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[32]</td>
                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Alias Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>NFT Address</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Address Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 16</strong> to denote an <i>NFT Address</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Address</td>
                                <td>ByteArray[20]</td>
                                <td>The raw bytes of the nft address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>Amount</td>
                <td>uint64</td>
                <td>The amount of IOTA tokens held by the output.</td>
            </tr>
            <tr>
                <td>Native Tokens Count</td>
                <td>uint16</td>
                <td>The number of native tokens held by the output.</td>
            </tr>
            <tr>
                <td valign="top">Native Tokens <code>optAnyOf</code></td>
                <td colspan="2">
                    <details>
                        <summary>Native Token</summary>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Token ID</td>
                                <td>ByteArray[38]</td>
                                <td>
                                    Identifier of the native token.
                                </td>
                            </tr>
                            <tr>
                                <td>Amount</td>
                                <td>uint256</td>
                                <td>
                                Amount of native tokens of the given <i>Token ID</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
            <tr>
                <td>NFT ID</td>
                <td>ByteArray[20]</td>
                <td>Unique identifier of the NFT, which is the BLAKE2b-160 hash of the <i>Output ID</i> that created it.<i> NFT Address = NFT Address Type || NFT ID</i></td>
            </tr>
            <tr>
                <td>Immutable Metadata Length</td>
                <td>uint32</td>
                <td>Length of the following <i>Immutable Metadata</i> field in bytes.</td>
            </tr>
            <tr>
                <td>Immutable Metadata</td>
                <td>ByteArray</td>
                <td>Binary metadata attached immutably to the NFT.</td>
            </tr>
            <tr>
                <td>Blocks Count</td>
                <td>uint16</td>
                <td>The number of feature blocks following.</td>
            </tr>
            <tr>
                <td valign="top">Blocks <code>atMostOneOfEach</code></td>
                <td colspan="2">
                    <details>
                        <summary>Sender Block</summary>
                        <blockquote>
                            Identifies the validated sender of the output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 0</strong> to denote a <i>Sender Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td valign="top">Sender <code>oneOf</code></td>
                                <td colspan="2">
                                    <details>
                                        <summary>Ed25519 Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[32]</td>
                                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>Alias Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[20]</td>
                                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                                            </tr>
                                        </table>
                                    </details>
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Issuer Block</summary>
                        <blockquote>
                            Identifies the validated issuer of the NFT output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 1</strong> to denote an <i>Issuer Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td valign="top">Issuer <code>oneOf</code></td>
                                <td colspan="2">
                                    <details>
                                        <summary>Ed25519 Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 0</strong> to denote an <i>Ed25519 Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[32]</td>
                                                <td>The raw bytes of the Ed25519 address which is a BLAKE2b-256 hash of the Ed25519 public key.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>Alias Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 8</strong> to denote an <i>Alias Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[20]</td>
                                                <td>The raw bytes of the alias address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                                            </tr>
                                        </table>
                                    </details>
                                    <details>
                                        <summary>NFT Address</summary>
                                        <table>
                                            <tr>
                                                <td><b>Name</b></td>
                                                <td><b>Type</b></td>
                                                <td><b>Description</b></td>
                                            </tr>
                                            <tr>
                                                <td>Address Type</td>
                                                <td>uint8</td>
                                                <td>
                                                    Set to <strong>value 16</strong> to denote an <i>NFT Address</i>.
                                                </td>
                                            </tr>
                                            <tr>
                                                <td>Address</td>
                                                <td>ByteArray[20]</td>
                                                <td>The raw bytes of the nft address which is a BLAKE2b-160 hash of the outputID that created it.</td>
                                            </tr>
                                        </table>
                                    </details>
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Return Amount Block</summary>
                        <blockquote>
                            Defines the amount of IOTAs that have to be returned to <i>Sender</i>.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 2</strong> to denote a <i>Return Amount Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Return Amount</td>
                                <td>uint64</td>
                                <td>
                                    Amount of IOTA tokens the consuming transaction should deposit to the address defined in <i>Sender Block</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Timelock Milestone Index Block</summary>
                        <blockquote>
                            Defines a milestone index until which the output can not be unlocked.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 3</strong> to denote a <i>Timelock Milestone Index Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Milestone Index</td>
                                <td>uint32</td>
                                <td>
                                    The milestone index starting from which the output can be consumed.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Timelock Unix Block</summary>
                        <blockquote>
                            Defines a unix time until which the output can not be unlocked.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 4</strong> to denote a <i>Timelock Unix Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Unix Time</td>
                                <td>uint32</td>
                                <td>
                                    Unix time (seconds since Unix epoch) starting from which the output can be consumed.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Expiration Milestone Index Block</summary>
                        <blockquote>
                            Defines a milestone index until which only <i>Address</i> is allowed to unlock the output. After the milestone index, only <i>Sender</i> can unlock it.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 5</strong> to denote a <i>Expiration Milestone Index Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Milestone Index</td>
                                <td>uint32</td>
                                <td>
                                    Before this milestone index, <i>Address</i> is allowed to unlock the output, after that only the address defined in <i>Sender Block</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Expiration Unix Block</summary>
                        <blockquote>
                            Defines a unix time until which only <i>Address</i> is allowed to unlock the output. After the expiration time, only <i>Sender</i> can unlock it.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 6</strong> to denote a <i>Expiration Unix Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Unix Time</td>
                                <td>uint32</td>
                                <td>
                                    Before this unix time, <i>Address</i> is allowed to unlock the output, after that only the address defined in <i>Sender Block</i>.
                                </td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Metadata Block</summary>
                        <blockquote>
                            Defines metadata (arbitrary binary data) that will be stored in the output.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 7</strong> to denote a <i>Metadata Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Data Length</td>
                                <td>uint32</td>
                                <td>
                                    Length of the following data field in bytes.
                                </td>
                            </tr>
                            <tr>
                                <td>Data</td>
                                <td>ByteArray</td>
                                <td>Binary data.</td>
                            </tr>
                        </table>
                    </details>
                    <details>
                        <summary>Indexation Block</summary>
                        <blockquote>
                            Defines an indexation tag to which the <i>Metadata Block</i> will be indexed. Creates an indexation lookup in nodes.
                        </blockquote>
                        <table>
                            <tr>
                                <td><b>Name</b></td>
                                <td><b>Type</b></td>
                                <td><b>Description</b></td>
                            </tr>
                            <tr>
                                <td>Block Type</td>
                                <td>uint8</td>
                                <td>
                                    Set to <strong>value 8</strong> to denote a <i>Indexation Block</i>.
                                </td>
                            </tr>
                            <tr>
                                <td>Indexation Tag Length</td>
                                <td>uint32</td>
                                <td>
                                    Length of the following indexation tag field in bytes.
                                </td>
                            </tr>
                            <tr>
                                <td>Indexation Tag</td>
                                <td>ByteArray</td>
                                <td>Binary indexation data.</td>
                            </tr>
                        </table>
                    </details>
                </td>
            </tr>
        </table>
    </details>
</table>

### Additional Transaction Syntactic Validation Rules

#### Output Syntactic Validation

- `Amount` field must fulfill the dust protection requirements.
- `Native Token Count` must not be greater than `Max Native Token Count Per Output`.
- It must hold true that `0` ≤ `Blocks Count` ≤ `9`.
- <i>Blocks</i> must be sorted in ascending order based on their `Block Type`.
- Syntactic validation of all present feature blocks must pass.
- `Immutable Metadata Length` fields can not be greater than `MaxMetadataLength`.
- `Address` must not be the same as the NFT address derived from `NFT ID`.

### Additional Transaction Semantic Validation Rules

- Explicit `NFT ID`: `NFT ID` is taken as the value of the `NFT ID` field in the NFT output.
- Implicit `NFT ID`: When an NFT output is consumed as an input in a transaction and `NFT ID` field is zeroed out, take
  the BLAKE2b-160 hash of the `Output ID` of the input as `NFT ID`.
- For every non-zero explicit `NFT ID` on the output side there must be a corresponding NFT on the input side. The
  corresponding NFT has the explicit or implicit `NFT ID` equal to that of the NFT on the output side.

#### Consumed Outputs
- The unlock block of the input corresponds to `Address` field and the unlock is valid.
- The unlock is valid if and only if all feature blocks present in the output validate.
- When a consumed NFT output has a corresponding NFT output on the output side, `Immutable Metadata Length` and
  `Immutable Data` fields are not allowed to change.
- When a consumed NFT output has an `Issuer Block` and a corresponding NFT output on the output side, `Issuer Block` is
  not allowed to change.

#### Created Outputs
- When `Issuer Block` is present in an output and explicit `NFT ID` is zeroed out, an input with `Address` field that
  corresponds to `Issuer` must be unlocked in the transaction.
- Feature block imposed transaction validation criteria must be fulfilled.

### Notes
- It would be possible to have two-step issuer verification: First NFT is minted, and then metadata can be immutably
  locked into the output. The metadata contains an issuer public key plus a signature of the unique `NFT ID`. This way
  a smart contract chain can mint on behalf of the user, and then push the issuer signature in a next step.

## Unlocking Chain Script Locked Outputs

Two of the introduced output types ([Alias](#alias-output), [NFT](#nft-output)) implement the so-called UTXO chain
constraint. These outputs receive their unique identifiers upon creation, generated by the protocol, and carry it
forward with them through transactions until they are destroyed. These unique identifiers (`Alias ID`, `NFT ID`) also
function as global addresses for the state machines, but unlike `Ed25519Addresses`, they are not backed by private keys
that could be used for signing. The rightful owners who can unlock these addresses are defined in the outputs
themselves.

Since such addresses are accounts in the ledger, it is possible to send funds to these addresses. The unlock mechanism
of such funds is designed in a way that **proving ownership of the address is reduced to the ability to unlock the
corresponding output that defines the address.**

### Alias Locking & Unlocking

A transaction may consume a (non-alias) output that belongs to an <i>Alias Address</i> by also consuming (and thus
unlocking) the alias output with the matching `Alias ID`. This serves the exact same purpose as providing a signature
to unlock a <i>SimpleOutput</i>.

On protocol level, alias unlocking is done using a new unlock block type, called **Alias Unlock Block**.

<details>
<summary>Alias Unlock Block</summary>
<blockquote>
        Points to the unlock block of a consumed alias output.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Unlock Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 2</strong> to denote a <i>Alias Unlock Block</i>.
        </td>
    </tr>
    <tr>
        <td>Alias Reference Unlock Index</td>
        <td>uint16</td>
        <td>
            Index of input and unlock block corresponding to an alias output.
        </td>
    </tr>
</table>

This unlock block is similar to the <i>Reference Unlock Block</i>. However, it is valid if and only if the input of the
transaction at index `Alias Reference Unlock Index` is an alias output with the same `Alias ID` as the `Address` field
of the to-be unlocked output.

Additionally, the <i>Alias Unlock Blocks</i> must also be ordered to prevent circular dependencies:

If the i-th *Unlock Block* of a transaction is an *Alias Unlock Block* and has `Alias Reference Unlock Index` set to k,
it must hold that i > k. Hence, an <i>Alias Unlock Block</i> can only reference an *Unlock Block* (unlocking the
corresponding alias) at a smaller index.

For example the scenario where `Alias A` is locked to the address of `Alias B` while `Alias B` is in locked to the
address of `Alias A` introduces a circular dependency and is not well-defined. By requiring the *Unlock Blocks* to be
ordered as described above, a transaction consuming `Alias A` as well as `Alias B` can never be valid as there would
always need to be one *Alias Unlock Block* referencing a greater index.

### NFT Locking & Unlocking

`NFT ID` field is functionally equivalent to `Alias ID` of an alias output. It is generated the same way, but it can
only exist in NFT outputs. Following the same analogy as for alias addresses, NFT addresses are iota addresses that are
controlled by whoever owns the NFT output itself.

Outputs that are locked under `NFT Address` can be unlocked by unlocking the NFT output in the same transaction that
defines `NFT Address`, that is, the NFT output where `NFT ID = NFT Address`.

An **NFT Unlock Block** looks and behaves like an <i>Alias Unlock Block</i>, but the referenced input at the index must
be an NFT output with the matching `NFT ID`.

<details>
<summary>NFT Unlock Block</summary>
<blockquote>
        Points to the unlock block of a consumed NFT output.
</blockquote>
</details>

<table>
    <tr>
        <td><b>Name</b></td>
        <td><b>Type</b></td>
        <td><b>Description</b></td>
    </tr>
    <tr>
        <td>Unlock Block Type</td>
        <td>uint8</td>
        <td>
            Set to <strong>value 3</strong> to denote a <i>NFT Unlock Block</i>.
        </td>
    </tr>
    <tr>
        <td>NFT Reference Unlock Index</td>
        <td>uint16</td>
        <td>
            Index of input and unlock block corresponding to an NFT output.
        </td>
    </tr>
</table>

An *Alias Unlock Block* is only valid if the input in the transaction at index `NFT Reference Unlock Index` is the NFT
output with the same `NFT ID` as the `Address` field of the to-be unlocked output.

If the i-th *Unlock Block* of a transaction is an *NFT Unlock Block* and has `NFT Reference Unlock Index` set to k, it
must hold that i > k. Hence, an <i>NFT Unlock Block</i> can only reference an *Unlock Block* at a smaller index.


# Drawbacks
- New output types increase transaction validation complexity, however it is still bounded.
- Outputs take up more space in the ledger, UTXO database size might increase.
- It is possible to intentionally deadlock aliases and NFTs, however client side software can notify users when they
  perform such action. Deadlocked aliases and NFTs can not be unlocked, but this is true for any funds locked into
  unspendable addresses.
- Time based output locking conditions can only be evaluated after attachment to the Tangle, during milestone
  confirmation.
- IOTA ledger can only support hard-coded scripts. Users can not write their own scripts because there is no way
  currently to charge them based on resource usage, all IOTA transactions are feeless by nature.
- Aliases can be destroyed even if there are foundries alive that they control. Since only the controlling alias can
  unlock the foundry, such foundries and the supply of the tokens remain forever locked in the Tangle.
- Token schemes and needed supply control rules are unclear.

# Rationale and alternatives

The feeless nature of IOTA makes it inherently impossible to implement smart contracts on layer 1. A smart contract
platform shall not only be capable of executing smart contracts, but also to limit their resource usage and make users
pay validators for the used resources. IOTA has no concept of validators, neither fees. While it would technically be
possible to run EUTXO smart contracts on the layer 1 Tangle, it is not possible to properly charge users for executing
them.

The current design aims to combine the best of both worlds: Scalable and feeless layer 1 and  Turing-complete smart
contracts on layer 2. Layer 1 remains scalable because of parallel transaction validation, feeless because the bounded
hard-coded script execution time, and layer 2 can offer support for all kinds of virtual machines, smart contracts and
advanced tokenization use cases.

# Unresolved questions

- List of supported <i>Token Schemes</i> is not complete.
    - Deflationary token scheme
    - Inflationary token scheme with scheduled minting
    - etc.
- Adapt the current congestion control, i.e. *Message PoW*, to better match the validation complexity of the different
  outputs and types.