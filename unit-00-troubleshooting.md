---
title: "Unit 0 troubleshooting"
sidebar:
  label: "Unit 0 troubleshooting"
---

<span id="unit-0-troubleshooting" />


[← Unit 0 — First Tiny-LM Run](/llm-engineering-course-pages/unit-00) · [Diagnostic & learning path →](/llm-engineering-course-pages/diagnostic) · [Glossary](/llm-engineering-course-pages/glossary)

Run the diagnostic first and fix the first failing check:

```
python examples/diagnose_unit0.py
```

| Symptom | Meaning | Recovery |
| --- | --- | --- |
| `python: command not found` | Python is not on this shell's path | Try `python3`; install Python 3.11+ if both fail |
| `No module named llm_course` | environment is inactive or package was not installed | Activate `.venv`, then run `pip install -e ".[dev,docs]"` |
| install appears slow | PyTorch is a large dependency | Keep the terminal open; wait for the final success/error line |
| requested MPS/CUDA is unavailable | accelerator is unsupported or not visible to PyTorch | rerun with `--device cpu`; this is the full required route |
| permission error under `artifacts/` | current directory is not writable | use `--artifact-dir` with a writable project-local directory |
| checkpoint format/vocabulary error | assets came from another course version | run with a fresh directory such as `--artifact-dir artifacts/unit0-clean` |
| `text must not be empty` | a language model needs at least one context token | provide a non-empty `--prompt` |
| prompt contains `?` unexpectedly | unsupported characters map to the unknown token | use lowercase letters and the listed punctuation for this first model |
| sampled runs differ | seed or command differs | use the same `--seed`, prompt, boost, temperature, and course commit |

## Backend isolation [#backend-isolation]

Check the mandatory CPU route independently:

```
python examples/diagnose_unit0.py --device cpu
python examples/run_unit0.py --device cpu
```

If CPU passes but MPS/CUDA fails, record backend, PyTorch version, command, and
full error. Continue the lesson on CPU; acceleration is optional.

## Evidence for a useful bug report [#evidence-for-a-useful-bug-report]

Include:

1. operating system and Python version;
2. output of `python examples/check_hardware.py --skip-benchmark`;
3. output of `python examples/diagnose_unit0.py`;
4. exact command and first complete traceback;
5. whether `--device cpu` succeeds.

Never attach generated checkpoints, company data, credentials, or personal
information.

[← Unit 0 — First Tiny-LM Run](/llm-engineering-course-pages/unit-00) · [Diagnostic & learning path →](/llm-engineering-course-pages/diagnostic) · [Glossary](/llm-engineering-course-pages/glossary)
