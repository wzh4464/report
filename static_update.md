{====================
Input
====================
You are a secure code auditor.
Analyze the given code snippet and determine if it exhibits a vulnerability that belongs to one of the following high-level CWE categories:

1. Improper Access Control (CWE-284)
2. Improper Interaction Between Multiple Correctly-Behaving Entities (CWE-435)
3. Improper Control of a Resource Through its Lifetime (CWE-664)
4. Incorrect Calculation (CWE-682)
5. Insufficient Control Flow Management (CWE-691)
6. Protection Mechanism Failure (CWE-693)
7. Incorrect Comparison (CWE-697)
8. Improper Check or Handling of Exceptional Conditions (CWE-703)
9. Improper Neutralization (CWE-707)
10. Improper Adherence to Coding Standards (CWE-710)
11. Others
12. Correct (no vulnerability)

Instructions:
- Thoroughly examine the code snippet for any signs of vulnerability.
- Select exactly one category.
- Clearly state why this category is applicable (or why the code is correct), with preference for "Others" over speculative assignment and "Correct" only if no vulnerability is evident.

Guidance:
- For confusion pairs (e.g., CWE-703 vs CWE-710), discern based on evidence: choose CWE-703 for mishandled error cleanup or propagation, and CWE-710 for coding standards issues without concrete fault.
- If protection mechanisms are present but failing, opt for CWE-693; if access control is missing or incorrectly enforced, opt for CWE-284.

Input code:
{{INPUT_CODE}}

Output format:
Category: [CWE Number and Name]
Reason: [Short justification]}