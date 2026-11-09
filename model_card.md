# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts. It is a teaching tool, not something to point at a real codebase unattended.

---

## 2) How does it work?

`BugHoundAgent.run()` runs five steps in order:

1. **Plan:** logs that a scan and fix workflow is starting. There is no real planning logic.
2. **Analyze:** finds issues. With a Gemini client it sends `analyzer_system.txt` and `analyzer_user.txt` and expects a JSON array of `{type, severity, msg}`. If the reply doesn't parse, it retries once with a stricter "JSON only" reminder. If that fails too, or the API call raises, it falls back to the heuristic analyzer.
3. **Act:** proposes a fix. With Gemini it asks for the full rewritten file. An empty reply or an API error falls back to the heuristic fixer. If no issues were found, the original code is returned unchanged.
4. **Test:** `assess_risk()` scores the fix from 0 to 100 and maps the score to low, medium or high.
5. **Reflect:** auto-fix is allowed only if the level is `low`, no issue is High severity, and the fixed code still compiles. Otherwise the tool recommends human review.

**Heuristics (offline, or as fallback):**
- Analyzer: `"print(" in code` gives Low, a bare `except:` regex gives High, and the string `TODO` gives Medium.
- Fixer: replaces `print(` with `logging.info(` and prepends `import logging`. It also rewrites `except:` to `except Exception as e:`.

**Gemini (when a key is set):** the model does the open-ended analysis and the rewrite. Risk scoring is always done by plain Python rules, never by the model.

---

## 3) Inputs and outputs

**Inputs:** the four files in `sample_code/`. They are all short single-function scripts:

| File | Shape |
|---|---|
| `cleanish.py` | 5 lines, already uses `logging` |
| `print_spam.py` | function with 3 `print()` calls |
| `flaky_try_except.py` | `open`/`read`/`close` inside a bare `except:` |
| `mixed_issues.py` | TODO comment, a `print`, and a bare `except:` around a division |

**Outputs, heuristic mode (run offline on each file):**

| File | Issues found | Score / level | Auto-fix? |
|---|---|---|---|
| `cleanish.py` | none | 100 / low | yes (nothing to change) |
| `print_spam.py` | Code Quality (Low) | 95 / low | **yes** |
| `flaky_try_except.py` | Reliability (High) | 55 / medium | no |
| `mixed_issues.py` | Code Quality (Low), Reliability (High), Maintainability (Medium) | 30 / high | no |

The fixes proposed were `print` to `logging.info`, `except:` to `except Exception as e:` with a comment, and a prepended `import logging`.

**Outputs, Gemini mode:** I ran `mixed_issues.py` and `print_spam.py` through Gemini. The only trace I kept is for `mixed_issues.py`. Gemini's analyzer reply was not parseable JSON, so analysis fell back to heuristics and found 3 issues. The fixer then ran in LLM mode, and the risk step reported **score 0 / high**, so the fix was not auto-applied. In `assess_risk`, a score of exactly 0 comes from the "No fix was produced" branch, so the fixer returned empty output on that run.

---

## 4) Reliability and safety rules

**Rule A: "Return statements may have been removed" (−30)**
`if "return" in original_code and "return" not in fixed_code`
- *Why it matters:* an LLM rewrite that drops `return` changes what the function gives its caller.
- *False positive:* the word `return` appears in a comment or string in the original, and a legitimate cleanup removes it. The fix is penalized for nothing.
- *False negative:* it is a substring check on the whole file. If any `return` survives, even in a different function, the check passes. A fix that deletes `return x / y` from the function that matters is not caught if another function still returns. It also misses a `return` whose value was changed.

**Rule B: "Fixed code is much shorter than original" (−20 if under 50% of the lines)**
- *Why it matters:* it catches over-editing, where the model deletes logic or returns a stub.
- *False positive:* a legitimate cleanup, such as deleting a large block of dead code or comments, trips it.
- *False negative:* a model can destroy behavior without changing the line count. It can also rewrite everything at the same length. The rule only looks at size, not at what changed.

**Rule C: severity-based deductions and the auto-fix gate (High −40, Medium −20, Low −5; autofix only if low and no High issue)**
- *Why it matters:* risky issues force a human to look, whatever the fix looks like.
- *False positive:* a High issue is always blocked, even when the fix is a trivial one-liner. `flaky_try_except.py` is a case where the fix is simple but is still blocked.
- *False negative:* the score uses the *issue's* severity, not the *fix's* risk. `print_spam.py` is Low severity, so it scores 95 and auto-applies, even though the heuristic fix breaks the code (see section 5).

**Rule D (added by me): the fixed code must compile**
`compile(fixed_code)` runs only if the original compiled. A fix that doesn't compile takes −60 and can never auto-apply.
- *False positive:* very few, because it only fires on real SyntaxErrors. If the original was already broken the rule is skipped.
- *False negative:* it only checks that the code is valid Python, not that it behaves the same. Both `logging.info("Hello", name)` (a runtime error) and a renamed variable pass.

---

## 5) Observed failure modes

1. **Heuristic fixer produces code that fails at runtime, and the guardrail auto-approves it.**
   `print_spam.py` contains `print("Hello", name)`. The fixer turns it into `logging.info("Hello", name)`. That is a bug: `logging.info` treats `name` as a %-format argument, and logging fails with "not all arguments converted". Because the only issue is Low severity, the score is 95 and `should_autofix` is True. This is the worst result, because it looks safe and isn't.

2. **Heuristic fix produces code that doesn't compile.**
   A file that starts with `from __future__ import annotations` and has one `print()` gets `import logging` prepended, which pushes the future import off the top and makes the file a SyntaxError. The old scorer approved it at 95. I fixed this with Rule D and added `test_unparseable_heuristic_fix_is_not_autofixed`. I first used `ast.parse` for the check, but it didn't catch this, because the future-import check runs in the compiler and not the parser. That's why the guardrail uses `compile()`.

3. **Missed issue (heuristic analyzer).**
   In `flaky_try_except.py` the heuristics flag only the bare `except:`. They miss that `f.close()` is skipped if `f.read()` raises (a resource leak, which should use `with`). They also miss that the handler silently returns `None`, so callers can't tell "missing file" from "empty file". The "fix" changes `except:` to `except Exception as e:` and leaves `e` unused with a comment saying "log or handle the error". That doesn't handle anything, it only changes the syntax.

4. **Format failure (Gemini mode).**
   On `mixed_issues.py` Gemini's analyzer reply wasn't valid JSON, even though the prompt says "Return ONLY valid JSON". The agent fell back to heuristics, which worked, but it means Gemini's analysis was thrown away. I added one retry with a stricter prompt. In my runs it made no visible difference to the outcome.

---

## 6) Heuristic vs Gemini comparison

I ran all four files in heuristic mode. I ran only `mixed_issues.py` and `print_spam.py` in Gemini mode, and I only kept the trace for `mixed_issues.py` (see section 3). So this comparison is limited:

- **What Gemini detected that heuristics did not:** I can't say. In the run I kept, Gemini's analysis was discarded because its output was unparseable.
- **What heuristics caught consistently:** all three patterns (`print(`, bare `except:`, `TODO`) every time, with no variation between runs. They are also fully predictable, which makes them easy to test offline with `client=None`.
- **Fixes:** heuristic fixes are mechanical and small, but sometimes wrong, as in `logging.info("Hello", name)`. The Gemini fixer is asked for a full rewrite, which carries more over-editing risk. That is the scenario the "shorter than original" and "return removed" rules guard against.
- **Did the risk scorer match my intuition?** Mostly for the bare-except cases. `flaky_try_except.py` (medium) and `mixed_issues.py` (high) match. It did not match for `print_spam.py`, which scored 95 and auto-applied despite a broken fix. The scorer measures how many issues were found and how big the diff is, not whether the fix is correct.

---

## 7) Human-in-the-loop decision

**Scenario:** the fix changes the **behavior of control flow or error handling**, for example when a bare `except:` that returned a fallback value gets rewritten. In `mixed_issues.py`, `except: return 0` hides the ZeroDivisionError and returns 0. Any automatic rewrite there changes what callers see, so a human should decide it.

- **Trigger:** the original code has a bare `except:` (or `except Exception:`) whose body contains `return`, `pass` or `continue`, and the fix modifies that block. The check could be an AST walk over `ExceptHandler` nodes in the original and fixed code.
- **Where:** in `assess_risk` in `risk_assessor.py`, next to the other auto-fix gating. It needs both original and fixed code, which are both already available there. It should force `should_autofix = False` and add a reason string. The UI then needs no changes, because it already displays `reasons`.
- **Message to show:** *"This fix changes how errors are handled (a bare `except:` that returns a fallback value). The fallback behavior is part of your program's contract, so BugHound won't apply it automatically. Please review the suggested change and decide what should happen on failure."*

---

## 8) Improvement idea

**Make the `print` fixer skip calls it can't convert safely, and re-run the compile check on its result.**

Change the heuristic: use `\bprint\(` instead of the plain substring in both the detector and the replacement, and only rewrite calls with a single argument and no `file=`, `sep=` or `end=`. Calls with several arguments are still reported, but are left alone with a note that they need manual conversion.

This fixes both the `blueprint(` to `bluelogging.info(` rename and the `logging.info("Hello", name)` runtime bug. It is a small regex change, about 15 minutes of work. It's measurable: add a test that runs `print_spam.py` and asserts that `"logging.info(\"Hello\", name)"` doesn't appear in the fixed code, or that the multi-argument case is not auto-fixed. Today that test would fail, because `print_spam.py` scores 95 and auto-applies a broken fix.
