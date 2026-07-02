# Choose Your Architecture

`This module explains the three ways to build an AI auditor`: 

01. **single-pass**,
02.  **multi-phase pipeline**, 
03. **multi-agent system**.

 The recommended choice is the **multi-phase pipeline** because it gives better precision and is easier to trust for real security audits.

## Why multi-phase?
- **Recon**: understand the codebase.
- **Detect**: find possible bugs.
- **Verify**: confirm if the issue is really exploitable.
- **Report**: show only verified findings.

## Why this is useful
- Reduces false positives.
- Makes audits more structured.
- Easier to improve over time.
- Better for deep manual review, not just fast scanning.

## Best fit
This approach is best for thorough audits where accuracy matters most, especially for an upcoming auditor building a trustworthy security tool.