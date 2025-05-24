# CHIP-2024-12 OP_EVAL: Function Evaluation

        Title: Function Evaluation
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2024-12-12
        Latest Revision Date: 2025-05-24
        Version: 1.0.2

## Summary

This proposal introduces the `OP_EVAL` operation, enabling Bitcoin Cash contract bytecode to be factored into reusable functions.

## Motivation & Benefits

- **Improved privacy and operational security** – By enabling contracts to hide unused code paths, vaulting systems can be designed to leak less information about security measures, current assets, and transaction history, improving [privacy and operational security](./alternatives.md#improved-privacy-and-operational-security-vs-status-quo) for end users.

- **Improved auditability** – Without reusable functions, existing contracts are forced to unnecessarily duplicate code segments, often with additional stack management operations that can obfuscate unintended behavior or complicate formal verification. By enabling function evaluation, this proposal enables contracts to be written using more succinct, auditable patterns.

- **Reduced transaction sizes** – By eliminating duplicated bytecode, contract lengths can be optimized for a wide variety of use cases: finite field arithmetic, pairing-based cryptography, zero-knowledge proof systems, homomorphic encryption, post-quantum cryptography, and other important applications for the future security and competitiveness of Bitcoin Cash.

## Deployment

Deployment of this specification is proposed for the May 2026 upgrade.

- Activation is proposed for `1763208000` MTP, (`2025-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1778846400` MTP, (`2026-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Technical Specification

The `OP_EVAL` opcode is defined at codepoint `0x62` (`98`) with the following behavior:

1. Pop the top item from the stack to interpret as bytecode.<sup>1</sup>
2. If the popped item does not have executable permission, fail the transaction immediately else proceed
3. Preserve the active bytecode (A.K.A. `script`), instruction pointer (A.K.A. program counter or `pc`), and index of the last executed code separator (A.K.A. `pbegincodehash`) by pushing them to the top of the control stack.<sup>2</sup> (Note that this subjects `OP_EVAL` evaluations to the existing control stack depth limit of `100`.)
4. Reset the instruction pointer and last executed code separator, then evaluate the stack-provided bytecode as if it were the active bytecode.<sup>3</sup> If the bytecode is malformed (i.e. a push operation requires more bytes than are available in the remaining segment of bytecode to be parsed), error.
5. When the evaluation is complete<sup>4</sup>, restore the original bytecode, instruction pointer, and last executed code separator, then continue evaluation after the OP_EVAL instruction.<sup>5</sup>

The executable permission is a single bit of extra state attached to stack items, with the following rules:

1. All verbatim data push operations (codepoints `0x00` to `0x60`, except `0x50`) will result in a stack item with the executable bit set.
2. Stack operations (codepoints `0x6b` to `0x7d`, except `0x74`) will preserve the executable bit (e.g. swap, rot) or pass it on (e.g. dup).
3. All other operations will clear the bit of popped items, and not have it set on pushed items (e.g. cat, split, add).
To further clarify, the bit will be cleared even if the operation only touched but didn't actually modify the stack item, e.g. `<executable data> OP_0 OP_CAT` will lose the executable bit.
Results of introspection opcodes will NOT have the executable permission set.

#### Clarifications

1. If the stack is empty, error. If the top stack item has a length of zero: error if the control stack depth limit of `100` would be exceeded by a non-empty evaluation; otherwise, continue after the `OP_EVAL` instruction.
2. Note that this requires the control stack to be capable of holding a new **stack frame** data type to preserve the state of the parent evaluation.
3. `OP_EVAL`ed bytecode otherwise shares the context of the parent evaluation:
   1. Evaluations performed by `OP_EVAL` can be understood as efficient function calls: the evaluation may modify the stack and alternate stack without limitation, and control stack usage (e.g. `OP_IF/OP_END_IF` or further `OP_EVAL`s) within the evaluation remains restricted by the 100-item depth limit.
   2. Executed `OP_CODESEPARATOR` operations record the index of the current instruction pointer (A.K.A. `pc`) within the `OP_EVAL`ed bytecode as the last executed code separator (A.K.A. `pbegincodehash`).
   3. The `OP_ACTIVEBYTECODE` operation produces the serialization of the active bytecode beginning from the last executed code separator.
   4. In signature operations, the covered bytecode includes only the active bytecode beginning from the last executed code separator (or if none have been executed in current evaluation, the full active bytecode).
4. For an OP_EVAL evaluation to complete successfully, the top element remaining on the control stack must be a stack frame to resume after the evaluation. (E.g. Validation fails if an `OP_IF` within the evalation is not resolved by a matching `OP_ENDIF` within the same evaluation.)
5. After an active bytecode has been fully evaluated, the next stack frame is resumed (restoring its bytecode, instruction pointer, and last executed code separator values) until all stack frames are complete (the control stack is empty) or the evaluation has produced some error. Note that two or more `OP_EVAL` operations may be resolved within the same virtual machine step (e.g. if a child `OP_EVAL` is the final instruction of a parent `OP_EVAL` evaluation).

## Rationale

- [Appendix: Rationale &rarr;](rationale.md#rationale)
  - [Preservation of Alternate Stack](#preservation-of-alternate-stack)
  - [Preservation of `OP_CODESEPARATOR` Support](#preservation-of-op_codeseparator-support)
  - [Non-Impact on Performance or Static Analysis](#non-impact-on-performance-or-static-analysis)
    - [Estimation of Contract Validation Performance](#estimation-of-contract-validation-performance)

## Evaluations of Alternatives

- [Appendix: Evaluations of Alternatives &rarr;](alternatives.md#evaluation-of-alternatives)
  - [Status Quo: Sidecar Inputs ("Emulated `OP_EVAL`")](#status-quo-sidecar-inputs-emulated-op_eval)
    - [Improved Privacy and Operational Security vs. Status Quo](#improved-privacy-and-operational-security-vs-status-quo)
    - [Improved Efficiency and User Experience vs. Status Quo](#improved-efficiency-and-user-experience-vs-status-quo)
    - [Notes](#notes)
  - [BIP12 `OP_EVAL`](#bip12-op_eval)
  - [Output-Level Function Annex](#output-level-function-annex)
  - [Word Definition Operations (`OP_DEFINE` \& `OP_INVOKE`)](#word-definition-operations-op_define--op_invoke)
  - [OP_EXEC](#op_exec)

## Risk Assessment

- [Appendix: Risk Assessment &rarr;](risk-assessment.md#risk-assessment)
  - [Risks \& Security Considerations](#risks--security-considerations)
    - [User Impact Risks](#user-impact-risks)
      - [Reduced or Equivalent Node Validation Costs](#reduced-or-equivalent-node-validation-costs)
      - [Increased or Equivalent Contract Capabilities](#increased-or-equivalent-contract-capabilities)
    - [Consensus Risks](#consensus-risks)
      - [Full-Transaction Test Vectors](#full-transaction-test-vectors)
      - [Additional Performance Benchmarks](#additional-performance-benchmarks)
      - [`Chipnet` Preview Activation](#chipnet-preview-activation)
    - [Denial-of-Service (DoS) Risks](#denial-of-service-dos-risks)
      - [Node Performance Safety Margin](#node-performance-safety-margin)
    - [Protocol Complexity Risks](#protocol-complexity-risks)
      - [Support for Post-Activation Simplification](#support-for-post-activation-simplification)
      - [Evaluation of Alternatives](#evaluation-of-alternatives)
  - [Upgrade Costs](#upgrade-costs)
    - [Node Upgrade Costs](#node-upgrade-costs)
    - [Ecosystem Upgrade Costs](#ecosystem-upgrade-costs)
  - [Maintenance Costs](#maintenance-costs)
    - [Node Maintenance Costs](#node-maintenance-costs)
    - [Ecosystem Maintenance Costs](#ecosystem-maintenance-costs)

## Test Vectors

This proposal includes [a suite of functional tests and benchmarks](./vmb_tests/) to verify the performance of all operations within virtual machine implementations.

## Implementations

Please see the following implementations for examples and additional test vectors:

- C++:
  - [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/) – A professional, miner-friendly node that solves practical problems for Bitcoin Cash. [Merge Request !1937](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1937).
- **JavaScript/TypeScript**:
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Branch `next`](https://github.com/bitauth/libauth/tree/next).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Branch `next`](https://github.com/bitauth/bitauth-ide/tree/next).

## Feedback & Reviews

- [OP_EVAL CHIP Issues](https://github.com/bitjson/bch-eval/issues)
- [`CHIP 2024-12 OP_EVAL: Function Evaluation` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2024-12-op-eval-function-evaluation/1450)

## Changelog

This section summarizes the evolution of this document.

- **v1.0.1 – 2025-05-02**
  - Clarify description ([#1](https://github.com/bitjson/bch-eval/pull/1))
  - Add [Rationale](./rationale.md)
  - Add [Evaluation of Alternatives](./alternatives.md)
  - Add [Risk Assessment](./risk-assessment.md)
  - Commit latest test vectors
  - Link to BCHN implementation
- **v1.0.0 – 2024-12-12**
  - Initial publication

## Copyright

This document is placed in the public domain.
