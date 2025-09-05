# Rationale

This section documents design decisions made in this specification.

<details>

<summary><strong>Table of Contents</strong></summary>

- [Rationale](#rationale)
  - [Simple, Statically-Applicable Contract Length Optimizations](#simple-statically-applicable-contract-length-optimizations)
  - [Immutability of Function Bodies](#immutability-of-function-bodies)
  - [Deferred Parsing of Function Bodies](#deferred-parsing-of-function-bodies)
  - [Use of Stack-Based Parameters](#use-of-stack-based-parameters)
  - [Use of Positive Integer-Based Function Identifiers](#use-of-positive-integer-based-function-identifiers)
  - [Support for Skipping Function Identifiers](#support-for-skipping-function-identifiers)
  - [Opcode Naming: `OP_DEFINE` and `OP_INVOKE`](#opcode-naming-op_define-and-op_invoke)
  - [Preservation of Alternate Stack](#preservation-of-alternate-stack)
  - [Preservation of `OP_CODESEPARATOR` Support](#preservation-of-op_codeseparator-support)
  - [Non-Impact on Performance or Static Analysis](#non-impact-on-performance-or-static-analysis)
    - [Estimation of Contract Validation Cost](#estimation-of-contract-validation-cost)

</details>

## Simple, Statically-Applicable Contract Length Optimizations

In addition to enabling deeper optimizations by contract authors and context-aware development tooling, this proposal also enables significant contract length optimizations that are simple to apply and verify, including via static analysis: repeated, contiguous segments of 3 or more instructions may be factored into immutable function calls, reducing their per-location encoding cost to 2 bytes (or 3 bytes beyond the 17th-most-encoded segment)<sup>1</sup>. Such transformations can narrow program execution – by reducing contract length, reducing operation cost budget, increasing operation density, and/or exceeding depth/nesting limits – without risk of broadening program execution, e.g. causing otherwise-failing constructions to succeed.

Notably, this approach enables significant optimizations without deep support for Bitcoin Cash's stack-oriented virtual machine: valuable transformations do not require stack awareness or stack scheduling algorithms. By enabling useful optimizations at varying levels of efficiency, generality, context-awareness, and overall complexity – **this proposal simplifies and reduces barriers to creating or extending existing development, compilation, and auditing software for Bitcoin Cash**.

<small>

1. For example, a high-level contract compiler could scan compiled contracts for repeated patterns of instructions (respecting push operation boundaries), excluding or restricting some operations (e.g. `OP_DEFINE`, `OP_INVOKE`, `OP_ACTIVEBYTECODE`, and `OP_CODESEPARATOR`), then rank order them by final encoding frequency (for maximum efficiency in contracts with more than 17 such segments, allowing common functions to be specified with `OP_0 OP_INVOKE` through `OP_16 OP_INVOKE`). The ranking of segments may then be `OP_DEFINE`d as a "header" of functions prior to the optimized bytecode and each function `OP_INVOKE`d in its prior location(s). If the contract has been otherwise designed or audited for runtime safety, this transformation can likewise be proven safe:
   - For fail-safe designs – e.g. that cannot be "frozen" by malicious spend attempts (i.e. malicious inputs aren't naively internalized by a covenant, rendering expected code paths inaccessible to later-attempted spends) – expected code paths need only remain functional (easily verified for most contracts, where various VM limits and operation cost budget remain far in excess of practical usage; e.g. compilers and tooling can simply re-execute the contract's tests).
   - For fail-unsafe designs (if selected vs. a fail-safe design for length optimization, external system compatibility, or other design reasons) – verified upper and lower VM resource bounds (e.g. maximum stack depth, control stack depth, and operation cost as determined by higher-level language tooling, symbolic execution, or otherwise) can be adjusted with the transformation given the optimizer's knowledge of the invocations added and unlocking bytes saved.

</small>

## Immutability of Function Bodies

This proposal enforces the immutability of function bodies after definition – attempting to redefine a function identifier during evaluation triggers immediate validation failure. Immutable function bodies simplify contract security analysis and tooling, eliminating some potential contract bugs and malicious code path obfuscation techniques.

Alternatively, this proposal could allow dynamic redefinition of function identifiers, offering additional flexibility to contract authors. In this alternative, contracts with more than 17 defined functions (identifiable with `OP_0` through `OP_16`) could potentially be optimized to save an additional byte per invocation in some circumstances (e.g the 18th-most-used function is used more frequently, after some point in the computation, than one of the 17 functions occupying single-byte identifiers, such that swapping them saves bytes in the final encoded contract).

However, note that this optimization saves only 1 byte per _encoded_ `OP_INVOKE` reference (e.g. secondary invocations within loops already account for zero bytes in the encoded contract), so such optimizations seem unlikely to result in meaningful efficiency gains for the vast majority of contracts (while requiring non-trivial logic in contract compilers).

Additionally, in the mutable alternative, function definitions could also be more generally used as off-stack "variables". `OP_DEFINE` could behave as an inefficient "`OP_STORE`" (an "`OP_DEFINE`d variable" would require prefixing and concatenation with a correct push instruction), while `OP_INVOKE` would be essentially equivalent to "`OP_FETCH`". (Note that contract development tooling can also be designed to enforce discipline on usage of mutation – compilers could reasonably disallow mutation of function definitions while allowing a specific, easily auditable "variable reference" style of usage.)

However, this approach would produce significantly less optimal contract lengths than existing compilers: even without [well-optimized stack scheduling](https://users.ece.cmu.edu/~koopman/stack_compiler/index.html), naive stack-based data management techniques (e.g. deep `OP_PICK`s and `OP_ROLL`s) are generally much more byte-efficient than the 4+ bytes of overhead required to set such "variables" via `OP_DEFINE`.

Given the safety advantages of immutable function bodies, coupled with the impracticality and inconsequentiality of potential optimizations made possible by mutability, this proposal deems immutable function bodies to be most prudent.

## Deferred Parsing of Function Bodies

This proposal minimizes validation costs and consensus risks by only parsing function bodies when invoked by a matching `OP_INVOKE`.

By avoiding any inspection or attempted "pre-parsing" of the function body, the computational cost of `OP_DEFINE` is minimized, particularly for unused functions (i.e. the contract evaluation takes a code path in which the function is never invoked). This behavior ensures that `OP_DEFINE` has a very low, predictable cost, allowing greater usage by contract authors and minimizing the risk of implementation-specific consensus or denial-of-service vulnerabilities arising from faulty pre-parsing logic.

Alternatively, an "eager parsing" implementation would require all virtual machine (VM) implementations to separate instruction parsing from instruction evaluation such that the parsing step may be performed during `OP_DEFINE`. In addition to wasting parsing work for unused functions, this alternative would push performance-sensitive VM implementations toward more complex implementations of `OP_INVOKE` (or even the top-level "evaluation loop") in an attempt to re-use the parsing work performed during `OP_DEFINE`, with the increased complexity cascading across other VM behaviors (esp. signature checking and bytecode introspection operations).

Finally, note that by design, `OP_DEFINE` cannot fail due to the contents of a function body – empty (zero-length) or malformed (incomplete trailing push instruction) function bodies; this minimizes edge cases, allowing contracts to omit or defer case-specific logic, potentially reducing the operation cost of branch tables, precomputation tables, and similar constructions.

## Use of Stack-Based Parameters

This proposal's `OP_DEFINE` and `OP_INVOKE` operations adopt the existing pattern of all other non-push operations – and the defined functions themselves – by accepting parameters from the stack. This approach maximizes the power and flexibility of each operation, simplifies transpilation from other languages to Bitcoin Cash VM bytecode, reduces contract lengths, and avoids breaking modifications to bytecode parsing (which would require wider ecosystem software modifications beyond VM implementations).

For comparison, consider an alternative approach using multi-byte `OP_DEFINE` and `OP_INVOKE` opcodes, where each codepoint would require a new, specialized parsing behavior (similar to the modified parsing of push operations). In this alternative, upon encountering an `OP_DEFINE`, the parser might be expected to read a `CompactUint` (function identifier) followed by a standard push instruction containing the function body; for `OP_INVOKE`, the parser would read only the trailing function identifier.

This multi-byte alternative would allow savings of 1 byte per definition and invocation of the 18th through 127th defined function (where this proposal must switch to `OP_PUSHBYTES_1` from `OP_0`-`OP_16`), 2 bytes per definition and invocation of the 128th through 253rd defined function (where this proposal must switch to `OP_PUSHBYTES_2`), etc.

However, by preventing function definition and/or invocation from accepting data from the stack (i.e. directly using the results of previous computation), this alternative would complicate or prevent transpilation of complex computations from other languages (where no such limitations are imposed) to Bitcoin Cash VM bytecode. Instead, many branch table, precomputation table, and similar operation cost-reducing constructions would need to be "unrolled" into expensive conditional blocks (e.g. `OP_DUP <case> OP_EQUAL OP_IF...OP_ENDIF OP_DUP <case> ... OP_EQUAL`), wasting operation cost and significantly lengthening contracts.

Finally, the multi-byte alternative would break compatibility of non-executed bytecode parsing across all existing and historical software, requiring updates to many development libraries and tools (including Bitcoin Cash-specific and multi-network projects) beyond simple updates to opcode names (whereas for this proposal, such projects would merely misreport the new opcode names as `OP_RESERVED1` and `OP_RESERVED2`).

Given the advantages of the standard, stack-based approach – and the significant disadvantages of a parser-modifying approach – this proposal considers a stack-based approach to be most prudent.

## Format of Function Identifiers

This proposal requires that function identifiers be stack items of length `0` to `7`, inclusive. This range maximizes flexibility in contracts while allowing VM implementations to encode both the identifier’s length and value in a single 64‑bit key.

Integer-based function identifiers are particularly appropriate given the existing design of the VM: specialized, number pushing opcodes occupy 18 codepoints in the instruction set (`OP_1NEGATE` and `OP_0` to `OP_16`), enabling up to 18 unique function identifiers to be encoded using a single byte. However, function identifiers are not required to be minimally encoded numbers, avoiding numeric decoding overhead and enabling a wider variety of usage patterns – e.g. "namespaces" (like `0x00` through `0xff`), the single-byte `OP_1NEGATE`, and other identifiers which cannot be parsed as minimal VM numbers.

Outperforming this proposal's approach (in terms of byte-efficiency per encoded invocation) would require reserving a range of user-definable `OP_INVOKE0` to `OP_INVOKE16` (saving 1 byte per invocation). However, function definitions beyond that range would again require use of a generic `OP_INVOKE` (or generic multi-byte opcode range to specify through e.g. `OP_INVOKE999`). As single-byte codepoints are very limited, this proposal considers the two-codepoint `OP_DEFINE`/`OP_INVOKE` approach to be most prudent.

## Support for Skipping Function Identifiers

This proposal allows functions to be `OP_DEFINE`d in any order, simplifying contract development, maximizing contract safety, and minimizing contract lengths.

An alternative approach which e.g. requires each successive definition to occupy the next index in the [Function Table](./readme.md#function-table) would make contracts more brittle: minor changes to a contract could require re-indexing of many functions, complicating development and/or requiring additional development and audit tooling.

## Opcode Naming: `OP_DEFINE` and `OP_INVOKE`

While opcode names are not standardized by consensus (and conflicting names are already used in practice by ecosystem software), it's common for implementers to adopt the naming conventions recommended by CHIPs. This proposal recommends the names `OP_DEFINE` and `OP_INVOKE`.

For `OP_DEFINE`, potential options also include `OP_DECLARE`, `OP_FUNCTION`, `OP_WORD`, `OP_CONSTANT`, or partial words and concatenations like `OP_DEF`, `OP_DEFFUN`, `OP_DEFFUNC`, `OP_DEFWORD`, `OP_DEFINEFUNCTION`, `OP_DEFSUBROUTINE`, `OP_CONST`, etc.

This proposal selects `OP_DEFINE` to 1) avoid forcing development tooling adherence to the "function" terminology (this proposal uses "function" for wider accessibility, but Forth dialects typically use the term "word"), 2) maximize clarity (an "`OP_DECLARE`" could plausibly leave definition of the function body to another operation) and 3) minimize typographical mistakes (e.g. "define" is likely to be found in spell checking dictionaries).

For `OP_INVOKE`, potential options also include `OP_CALL`, `OP_RUN`, `OP_EXECUTE`, `OP_INTERPRET`, `OP_EVALUATE`, `OP_METHOD`, `OP_SUBROUTINE` or partial words and concatenations like `OP_CALLFUNC`, `OP_RUNFUNC`, `OP_EXEC`, `OP_EVAL`, etc.

This proposal selects `OP_INVOKE` to 1) minimize ambiguity in a variety of contexts ("call" and "run" are more common terms in many environments, with common uses unrelated to function invocation), 2) minimize ambiguity vs. similar concepts in other Forth dialects ("CALL", "INTERPRET", and "EXECUTE" might imply subtle differences which are not applicable to the Bitcoin Cash VM's combined parse-then-execute approach), and 3) minimize typographical mistakes (e.g. "invoke" is likely to be found in spell checking dictionaries).

## Preservation of Alternate Stack

Though not specified, the BIP12 `OP_EVAL` implementation [cleared the alternate stack](https://github.com/bitcoin/bitcoin/issues/729#issuecomment-3294453) at the end of each evaluation. At best, this behavior is useless: because each alternate stack item requires one byte to push (via `OP_TOALTSTACK`), any program which employed the stack clearing behavior could just as easily have replaced those bytes with equally or more efficient dropping operations (`OP_DROP`, `OP_2DROP`, or wider [stack clearing](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#increased-usability-of-multisig-stack-clearing)). Further, preventing functions from accepting or returning values in the alternate stack would limit their byte efficiency by requiring less efficient stack juggling strategies vs. the higher semantic bandwidth of passing both stacks.

Finally, clearing the alternate stack would increase the complexity of some [statically-applicable optimizations](##simple-statically-applicable-contract-length-optimizations), requiring either additional alternate stack-aware logic or exclusion of bytecode segments dealing with the alternate stack (`OP_TOALTSTACK` and `OP_FROMALTSTACK`).

## Preservation of `OP_CODESEPARATOR` Support

BIP12 `OP_EVAL` required implementations to specifically scan each evaluated script for instances of `OP_CODESEPARATOR`, rejecting the transaction if found. This needlessly increases implementation complexity and eliminates some potential value: [`OP_CODESEPARATOR` remains a useful feature](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#ongoing-value-of-op_codeseparator-operation) for certain use cases and has been preserved by past upgrades (e.g. [`OP_ACTIVEBYTECODE`'s support for `OP_CODESEPARATOR`](https://github.com/bitjson/bch-2022/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md#op_activebytecode-support-for-op_codeseparator)). This proposal avoids the additional implementation complexity and preserves the usefulness of `OP_CODESEPARATOR` within evaluations. See: [Should `OP_ACTIVEBYTECODE` encode the whole “call stack”?](https://bitcoincashresearch.org/t/chip-2024-12-op-eval-function-evaluation/1450/52#should-op_activebytecode-encode-the-whole-call-stack-6)

## Non-Impact on Performance or Static Analysis

The 2011 [BIP12 `OP_EVAL` proposal](https://github.com/bitcoin/bips/blob/master/bip-0012.mediawiki) was ultimately [abandoned](https://web.archive.org/web/20140420092207/https://bitcointalk.org/index.php?topic=46538.0;all#msg2256663) in favor of [BIP16 Pay to Script Hash (P2SH)](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki) due to uncertainty about `OP_EVAL`'s impact on the safety, validation performance, and "static analyzability" of Bitcoin Cash Virtual Machine (VM) bytecode<sup>1</sup>.

While some of these uncertainties have been addressed by [improvements vs. the BIP12 proposal](./alternatives.md#bip12-op_eval), others have been obviated by additional research and protocol upgrades in the intervening years.

### Estimation of Contract Validation Cost

While the Bitcoin VM initially had few limits, following the [2010 emergency patches](https://github.com/bitjson/bch-loops/blob/master/alternatives.md#timeline-of-emergency-patches), it could be argued that the expected runtime of a contract was (in principle) a function of the contract's length, i.e. long contracts take longer to validate than short contracts.

In practice, expensive operations (hashing and signature checking) have always dominated other operations by many orders of magnitude, but some developers [still](https://web.archive.org/web/20151210171935/https://github.com/bitcoin/bitcoin/issues/729) [considered](https://web.archive.org/web/20140416004759/https://bitcointalk.org/index.php?topic=58579.0;all) it potentially useful that contracts could "in principle" be somehow "reviewed" prior to execution, limiting wasted computing resources vs. "fully evaluating" the contract.

As is now highlighted by more than a decade of disuse in node implementations and other software: such "review" would necessarily optimize for the uncommon case (an invalid transaction from a misbehaving peer) by penalizing performance in the common case (standard transactions) **leading to worse overall validation performance and network throughput – even if validation cost weren't dominated by the most expensive operations**.

The [2018 restoration of disabled opcodes](https://upgradespecs.bitcoincashnode.org/may-2018-reenabled-opcodes/) further reduced the plausibility of non-evaluated contract analysis by reenabling opcodes that could unpredictably branch or operate on hashes (`OP_SPLIT`, bitwise operations, etc.). For example, the pattern `OP_HASH256 OP_1 OP_SPLIT OP_DROP OP_0 OP_GREATERTHAN OP_IF OP_HASH256 ...` branches based on the result of a hash, obviating any further attempt to inspect without computing the hash. Note that simply "counting" `OP_HASH256` operations here also isn't meaningful: valid contracts can rely on many hashing operations (e.g. Merkle trees), and the more performance-relevant [digest iteration count](https://github.com/bitjson/bch-vm-limits/blob/55b9a446eef4932627d387c622ef0fce38ec9512/readme.md#digest-iteration-count) of any evaluation depends on the precise input lengths of hashed stack items; the existence of hash-based branching implies that such lengths cannot be predicted without a vulnerability in the hashing algorithm.

Later, the [SigChecks (2020)](https://upgradespecs.bitcoincashnode.org/2020-05-15-sigchecks/) resolved [long-standing issues with the SigOps limit](https://upgradespecs.bitcoincashnode.org/2020-05-15-sigchecks/#motivation) by counting signature checks over the course of evaluation rather than by attempting to naively parse VM bytecode. Finally, the [VM limits (2025)](https://github.com/bitjson/bch-vm-limits) upgrade retargeted all VM limits to use similar density-based evaluation limits.

In summary, **fast validation is a fundamental and intentional feature of the VM itself** – critical for the overall throughput of the Bitcoin Cash network. Hypothetical "pre-validation" of contracts never offered improved performance, would have unnecessarily complicated all implementing software, and has been further obviated by multiple protocol upgrades in the intervening years.

## Non-Impact on Code Mutability or Code Injection

Following concerns about code mutability and injection, the CHIP maintainer conducted a systematic review of the topic as interpreted by various stakeholders. The review notes that:

- [Code mutation is explicitly disallowed by the Functions CHIP](https://bitcoincashresearch.org/t/chip-2025-05-functions-function-definition-and-invocation-operations/1576/20#code-mutation-is-explicitly-disallowed-by-the-functions-chip-1),
- [Native functions are trivial to use safely](https://bitcoincashresearch.org/t/chip-2025-05-functions-function-definition-and-invocation-operations/1576/20#native-functions-are-trivial-to-use-safely-2),
- [We already have delegation, it’s simple and useful](https://bitcoincashresearch.org/t/chip-2025-05-functions-function-definition-and-invocation-operations/1576/20#we-already-have-delegation-its-simple-and-useful-3), and
- [We already have “code that writes code”](https://bitcoincashresearch.org/t/chip-2025-05-functions-function-definition-and-invocation-operations/1576/20#we-already-have-code-that-writes-code-4).

A more detailed review of the "code that writes code" interpretation also provided [a practical example and explanation](https://bitcoincashresearch.org/t/chip-2025-08-functions-takes-2-3/1656/29#we-already-have-code-that-writes-code-2) in which such constructions improve privacy and efficiency today. It also notes that activation of the Loops CHIP would [make "code that writes code" even more practical](https://bitcoincashresearch.org/t/chip-2025-08-functions-takes-2-3/1656/29#loops-chip-would-make-the-above-discussion-irrelevant-3).

Further, in reviewing [Alternative: CHIP-2025-08 Functions (Takes 2 & 3)](./alternatives.md#alternative-chip-2025-08-functions-takes-2--3), it was found that function factories have negligible or positive impacts on practical contract security<sup>1</sup>, can simplify both compilers and contract algorithms<sup>2</sup>, and meaningfully improve validation performance (ultimately reducing transaction sizes) for opCost-limited contracts<sup>3</sup>.

<small>

1. [Reviewing risks: self-unpacking code - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2025-08-functions-takes-2-3/1656/48#reviewing-risks-self-unpacking-code-7)
2. [Mutation tracking makes contracts more complex, less efficient, and harder to audit - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2025-08-functions-takes-2-3/1656/48#mutation-tracking-makes-contracts-more-complex-less-efficient-and-harder-to-audit-11n)
3. [The problem with stack-based constants - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2025-08-functions-takes-2-3/1656/48#the-problem-with-stack-based-constants-12)

</small>
