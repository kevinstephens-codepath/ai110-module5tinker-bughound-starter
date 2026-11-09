# TF Notes for BugHound - Week 9
## Part 1 -
Chose mixed_issues.py for my sample.  
"Where BugHound decides: what's wrong, how to change it, and whether the change is safe." -   
It decides in analyze, and it checks in test.  
```
Agent trace
PLAN: Planning a quick scan + fix proposal workflow.

ANALYZE: Using LLM analyzer.

ANALYZE: LLM output was not parseable JSON. Falling back to heuristics.

ANALYZE: Found 3 issue(s).

ACT: Using LLM fixer.

TEST: Risk assessed as high (score=0).

REFLECT: Fix is not safe enough to auto-apply. Human review recommended.
```

> Q. Which step in the trace did the most work, and what made its output feel unreliable?  
> A. The trace that did the most work is the analyze step, it can loop while analyzing and is what ultimately decides what issues are present.

## Part 2 - 
Analyzed prompts, switched to Gemini mode, and ran mixed_issues and print_spam through it.  
I changed analyze to make one retry if the LLM response can't be parsed as JSON. The common transient failure is malformed output from LLM, therefore we can retry once and see if the LLM can correct the output.  
> Q. What did the agent reject or accept differently after your change?  
> A. It seemed to accept everything the same before and after.

## Part 3 -
Skimmed propse_fix for proper prompts
Looked over risk_assessor.py, and decided to tighten the auto-fix rule to only apply when the level is low AND no issue is high severity, as this helps block changes that could be more risky as an auto fix.

> Q. Which risk rule or fix constraint mattered most to the decision you changed?  
> A. Applying more caution to auto-fixing was what mattered to me so that's why I made that change.

## Part 4 -
Made one change to risk_assessor.py then added a test into test_agent_workflow.py

> Q. Which failure mode did your guardrail address, and what decision should the test protect?
> A. Failure mode: the heuristic fixer can turn valid code into code that no longer compiles, and the risk scorer used to approve it anyway. The scorer only looked at issue severity, line count and whether return disappeared. It never checked whether the fixed code was valid Python. In the test case, a file starting with from __future__ import annotations and containing one print() gets import logging prepended. The __future__ import is no longer at the top, so the file is a SyntaxError. The only issue found is Low severity, so the score stayed at 95 ("low") and the fix was marked safe to auto-apply. Decision the test protects: should_autofix must be False when the original code compiles and the proposed fix doesn't. Even if the fix looks minor and the score is otherwise high, a fix that doesn't compile must go to a human. The test asserts three things: - should_autofix is False -the level is not "low" - a reason mentions "does not parse", so the user can see why the fix was blocked

## Part 5 -

> Q. What is one failure mode or safety rule that the model card must capture, and why?  
> A.  