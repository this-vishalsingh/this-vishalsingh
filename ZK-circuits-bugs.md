### Vulnerabilities

V1. Under-Constrained: The most frequent vulnerability
in ZK circuits arises from insufficient constraints. This defi-
ciency causes the verifier to mistakenly accept invalid proofs,
thus undermining the system’s soundness or completeness.

V2. Over-Constrained: Although less common than under-
constrained issues, circuits can be over-constrained, leading
to the rejection of valid witnesses by honest provers or benign
proofs from honest verifiers. This issue stems from extra
constraints in the circuit, where legitimate solutions cannot
be proven or verified, leading to DoS issues. Nevertheless,
over-constrained bugs should not be confused with redundant
constraints that add no additional value but do not lead to any
issues other than introducing computational inefficiencies.

V3. Computational/Hints Error: Occurs when the computa-
tional part of a circuit is erroneous, often leading to complete-
ness issues where for correct inputs, the witness generation
either fails or produces wrong results. Note that completeness
issues can often be transient, meaning that you can fix the
underlying issue without having to update the circuit and re-
compute the prover and/or the verifier keys. Computational
issues may also result in soundness issues if the constraints
are applied using the same erroneous logic.

### Root Causes

R1. Assigned but not Constrained: A frequent issue in ZK
circuit design lies in distinguishing between assignments and
constraints. While constraints are mathematical equations
that must be satisfied for any given witness for the proof
to be valid, assignments allocate values to variables during
the witness generation process. The problem arises when
variables are assigned values based on correct computations
but lack corresponding constraints. This oversight can lead
the verifier to erroneously accept any value for these variables.


R2. Missing Input Constraints: Developers sometimes ne-
glect to apply constraints on input variables in reusable cir-
cuits. This omission occurs either (i) unintentionally or (ii)
because they anticipate these constraints will be enforced at a
different interface level (i.e., caller circuit or integration layer),
thus omitting the constraints for optimization reasons. How-
ever, the absence of clear documentation or fully constrained
inputs in these circuits can lead to severe vulnerabilities. Note
that this issue is particularly common in low-level circuit
DSLs, such as Circom, which lack user-defined types.


R3. Unsafe Reuse of Circuit: In ZK circuit design, particu-
larly when using DSLs like Circom, the practice of reusing
circuits (such as templates in Circom or gadgets in halo2)
can introduce vulnerabilities if not handled correctly. This
primarily occurs in two scenarios: (i) Implicit Constraints
in Sub-Circuits, occurred when circuits are reused without
appropriately constraining their inputs or outputs based on
the assumption that the user will apply these constraints on
call-site. (ii) Insecure Circuits Instantiation when circuits are
meant to be used for specific setups (e.g., specific curves).
An example is the Sign template from circomlib, which was
designed solely for the BN254 curve field.

R4. Wrong translation of logic into constraints: Translat-
ing computations for ZK circuits presents a unique set of
challenges, primarily due to the distinct programming model
of ZK circuits compared to traditional CPU-targeted code. A
significant issue arises when translating logic that involves
types and operations available in conventional programming
languages but are either absent or must be re-implemented
in ZK frameworks (e.g., fixed-point arithmetic). This often
leads to the need for creative but error-prone solutions, such
as using multiplexers for conditional logic. Developers might
inadvertently omit essential constraints or simplify the logic
to reduce the number of constraints, potentially missing criti-
cal corner cases. This can leave variables under-constrained,
allowing them to accept multiple or any values under certain
conditions, thus deviating from the developer’s intent and
introducing vulnerabilities.

R5. Incorrect Custom Gates: In implementations following
the TurboPLONK model, such as Halo2, circuit constraints
are defined by using custom gates applied to specific rows
and cells of a table that constitute the Plonkish representation
of the circuit. However, this approach can lead to bugs when
custom gates are incorrectly handled. Errors may arise from
inaccurately determining the appropriate offsets, resulting in
misalignment with the intended behavior of the circuit.

R6. Out-of-Circuit Computation Not Being Constrained:
Out-of-circuit computation refers to computations within the
code that do not directly impact the witness generation yet
play a crucial role in the overall functionality. In DSLs like
Circom, certain functions operate outside the circuit logic.
Similarly, in eDSLs, standard code (e.g., vanilla Rust) is used
for various computations that do not affect witness genera-
tion. For instance, computations like division, are typically
performed out-of-circuit to optimize circuit efficiency. This
method involves executing the computation externally and
then witnessing the result back into the circuit where it is
constrained to ensure correctness. Issues arise when these out-
of-circuit computations lead to missing assignments or when
constraints necessary for the circuit’s integrity are overlooked.
A specific manifestation of this issue is the boomerang issue,
where a variable, initially constrained within the circuit, is
temporarily moved out-of-circuit and then reintegrated with-
out the necessary reapplication of constraints.

R7. Arithmetic Field Error: Working with field arithmetic
in ZK circuits can be challenging, especially as developers
might overlook the nuances of computations within a finite
field. The most common issues in this context are arithmetic overflows and underflows. We categorize the primary types
of overflows and underflows in ZK circuits as follows: 
(i)
Native Field Arithmetic Over/Underflow, occurs when circuit
computations exceed the finite field’s limits, causing values to
loop back within the field’s range due to modulo arithmetic.
(ii) Overflows in Transformed Formats, risks of overflows
arise when numbers are transformed into bit representations
for specific operations (e.g., range checks) or for emulating
non-native arithmetics like fixed-point arithmetics. This is
particularly problematic when multiple bit representations
remain within the field’s overflow limits, leading to under-
constrained vulnerabilities.

R8. Bad Circuit/Protocol Design: Circuit design issues in
SNARKs often stem from fundamental flaws in how circuits
are conceptualized, potentially leading to unintended behav-
iors or the violation of protocol properties. A common man-
ifestation of this problem is the incorrect categorization of
variables – such as designating a variable as private when it
should be public. These issues can significantly impact the
functionality and security of the protocol.

R9. Other Programming Errors: This category encom-
passes a range of common programming errors that do not
fit neatly into other specific vulnerability categories but still
have significant implications for the integrity of SNARK sys-
tems. These errors include, but are not limited to, API misuse,
incorrect indexing in arrays, and logical errors within the com-
putational parts (i.e., witness generation) of the circuit that
are not directly related to constraint application.



