## Client Onboarding process 

1. Audit Preparation & Scope Definition

Before formally starting the audit, align with the client to:
Confirm the Codebase & Commit Hash: I identify the exact version of the code to audit, avoiding last-minute changes.

Review Documentation & Architecture: We assess high-level design choices, usage scenarios, and any existing developer docs.

Discuss Project Goals & Concerns: If needed, hold a “pre-audit call” to clarify any unique business logic or specific issues the client wants examined.

In some cases, we provide an “audit kit” that highlights missing tests or documentation, so the team can resolve these gaps before we start. This helps us focus on deeper security reviews.

2. Kick-Off & Audit Planning

Once everything is set, it's time to officially begin:
- Kick-Off Call: We meet with the client’s team to confirm final details, timeline, scope, and any nuances discovered from an initial code review.

- Audit Plan: Our internal lead creates an audit plan outlining the structure and responsibilities for each auditor. This ensures every part of the code is thoroughly reviewed in a systematic, consistent manner.

3. Code Review

At the heart of every OpenZeppelin audit, at least two security auditors are assigned to review the same codebase. This ensures that each line of code undergoes multiple sets of expert eyes, catching issues that might slip past a single reviewer.

Our core security review consists of two key phases:
1. Thorough Manual Analysis

- Line-by-Line Review: Our security engineers scrutinize every part of the code, looking for logical flaws, edge cases, and business logic errors.
- Access Control Checks: We confirm role and permission configurations are correctly implemented across all functions.
Custom Logic & Edge Cases: We pay special attention to project-specific features or assumptions that might introduce unforeseen vulnerabilities.

2. Strategic Use of Automated Tools

Static Analysis: Automated scanners flag common security patterns (e.g., integer overflows) that might otherwise slip through.

This synergy of human expertise and specialized tools helps us identify both high-level design flaws and common coding pitfalls, ensuring a robust audit that leaves no stone unturned.

4. Ongoing Communication & Check-Ins

Throughout the audit, we maintain regular check-ins 
with the client to:
Clarify Complex Areas: If any portion of the code raises questions, we discuss with the client’s dev team.
Flag Critical Findings Early: For critical or high-severity issues, we alert the client right away so they can begin planning fixes.

5. Reporting & Fix Review

Upon completion, we produce an initial private audit report, detailing:
All Found Issues: Clearly categorized by severity (Critical, High, Medium, Low, Informational). At OpenZeppelin, we are committed to excellence and it's in our mission to not only spot vulnerabilities but also provide suggestions on the code (Low and Informational issues) to improve the general robustness, quality and readability, ultimately reducing the chance of future bugs.
Recommendations: Step-by-step guidance on how to remediate each vulnerability.
Context & Rationale: Why a particular issue matters and how it could be exploited.
Methodology Summary (optional): An overview of our thorough, multi-layered approach for teams that require deeper insights into our assessment and risk analysis processes.

The client is entitled to one round of fix reviews provided each fix is in a dedicated pull request. While each fix has to be submitted as a separate PR for focused, isolated analysis, we always consider the entire codebase to prevent new issues. Once the client addresses these issues, we verify the solutions and finalize the report.

6. Retro & Knowledge Sharing

After delivering the initial report, our audit team devotes a “Retro Day” to:
Document Lessons Learned: Summarize any new attack vectors or notable approaches uncovered during the audit.

Expand Our Research Wiki: If we’ve discovered novel vulnerabilities, we add them to our internal knowledge base, ensuring our entire team stays updated.

We also gather feedback from the client to adapt and improve based on these insights ensuring our processes remain flexible, efficient, and aligned with each team’s unique needs.

7. Final Delivery & Optional Publication

We update the report with verified fixes and provide a final, polished PDF report. If desired, we can also coordinate a public audit release via the OpenZeppelin Blog to share results with the broader community, further demonstrating transparency and commitment to security.
