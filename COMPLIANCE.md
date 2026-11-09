# Compliance and Academic-Integrity Checklist

This file is a transparency aid, not a substitute for your programme's rules. It records what the repository does and does not do, so a reviewer can verify the work instead of guessing.

## What the repository guarantees

- **Supplied files are unmodified.** `generate_data.py` and `mock_llm.py` are the two files supplied with the brief and are used exactly as provided.
- **Synthetic data only.** No real customer, order, payment or support data is present. Emails and phone numbers in the tickets are generated placeholders.
- **No network calls and no secrets.** Every model call goes through the offline `mock_llm.py`. There is no API key, no `.env` file, no HTTP client, no hosted model and no paid service anywhere in the code.
- **No copied student code.** The Part A–D scripts, SQL and README were written for this repository. The only external references are public library documentation.
- **Privacy guardrail.** Ticket text is redacted for email, phone and order id before any prompt is built, and the code asserts that no pattern survives.
- **Reproducible.** Fixed seeds, pinned requirements and a numbered Reproduce section let a reviewer regenerate every checkpoint.
- **Limitations are stated.** The README says which results are illustrative, including the post-hoc Premium label used in the retention list.

## What this repository does not claim

It does **not** claim that no AI assistance was used to create it. That is a disclosure question for the submitter and their institution. It also does not claim that the synthetic findings generalise to real Instamart data.

## Required disclosure (complete this honestly)

Replace the placeholders before submission, or follow the exact wording your programme requires:

```
AI assistance: [yes / no]
Tools used: [for example: none, or the name of a coding assistant]
Scope of assistance: [for example: drafting code, debugging, documentation]
Human review: [your name] has read, run and can explain every part of this repository.
Institution policy reference: [course code / policy section]
```

If your programme **prohibits** AI assistance, do not submit this repository as your own work. Use it only as a reference and follow your institution's policy. If your programme **allows** AI assistance with disclosure, submit the completed disclosure above alongside the repository.

## Automated check

Run:

```bash
python src/compliance_check.py
```

It checks that the required files exist, that the supplied files are present, that `.gitignore` covers secrets and virtual environments, that no Python file imports a network or hosted-LLM client, and that no common secret pattern appears in the code. It prints `COMPLIANCE CHECKS PASSED` when all checks hold.

## Reviewer checklist

- [ ] `generate_data.py` and `mock_llm.py` are the supplied versions.
- [ ] `python generate_data.py` ends with the exact generator line.
- [ ] `python src/validate_part_a.py` ends with `ALL CHECKPOINTS PASSED`.
- [ ] `python src/compliance_check.py` ends with `COMPLIANCE CHECKS PASSED`.
- [ ] The disclosure above is completed and matches the institution's policy.
- [ ] The submitter can explain the cleaning rules, the K-means choice, the leakage controls, the retry logic, the routing rules and the refund loop without reading from a script.
