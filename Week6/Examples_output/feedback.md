# Counterexample — Guideline 2: Add algorithmic details when logic is complex

## 1. Rationale

Guideline 2 assumes adding _algorithmic_ detail (time complexity targets, specific techniques, preferred control flow) improves code generation quality when logic is complex.

However, some “complex” codegen tasks are complex due to **integration/contract concerns**, not algorithms (e.g., config schema, dynamic imports, plugin registries, error surfaces). In these cases, asking for “optimal time complexity” can push Copilot to optimize the wrong thing (caching/lookups) and produce lots of algorithmic commentary while the _real risks_ are validation, failure handling, and clear contracts.

This counterexample shows Guideline 2 can be misapplied: it doesn’t guide users on what to do when complexity is **architectural/I-O**, not computational.

---

## 2. Example Problem

**Task Description:** Implement and explain `plugin_loader.py` (loads JSON config, resolves module, imports plugins dynamically, builds a pipeline).

**Artifact Used:** `plugin_loader.py` (plugin loader + pipeline builder).

---

## 3. Documentation: Original Guideline Applied

### Prompt (Original — applying Guideline 2 too literally)

> Implement `plugin_loader.py` using an efficient algorithmic approach. Ensure optimal time complexity. Avoid redundant operations. Explain algorithmic design decisions clearly.

### Copilot Output (Original — full)

```text
Algorithmic design decisions and complexity

Key goals

Minimize redundant work when the loader is used repeatedly (avoid re-imports, re-validation, and rebuilding pipelines).
Keep per-input pipeline execution optimal (low constant factors).
Maintain simple, predictable behavior with bounded caches.
What I changed (efficient approach)

Caching plugin registry by module name:
Implemented @functools.lru_cache(maxsize=32) on load_plugins(module_name).
Effect: repeated loads for the same module reuse the validated registry without re-importing/calling factory repeatedly.
Complexity: first load O(R) to validate (R = number of registry entries); subsequent loads O(1) to fetch from cache (amortized).
Pipeline construction cache:
Added a small in-function cache keyed by (module_name, tuple(steps)) inside init_pipeline.
Effect: repeated init_pipeline calls for the same module+steps return the same pipeline object without re-validation or rebuild.
Complexity: cache lookup O(1); first build cost O(k + m) (k = number of steps, m = registry size for validation), subsequent retrieval O(1).
Pre-resolving plugin callables:
build_pipeline now pre-resolves [registry[step] for step in steps] and the pipeline iterates over these callables directly.
Effect: avoids dict lookups on every pipeline invocation; reduces per-step constant-time overhead.
Complexity: building callables list O(k), pipeline execution O(k) per input but with lower constant factor (one function call per step).
Validation batched and efficient:
Missing-step detection uses a single comprehension/list to find missing steps (one pass).
Registry validation runs once per registry load (cached) instead of every pipeline build.
Time complexity summary

load_config(path): O(C) to read JSON (C = config file size).
load_plugins(module_name) first call: O(m) to verify registry entries; subsequent calls: O(1) via LRU cache.
build_pipeline(registry, steps) first call: O(k) to validate and build callables (k = number of steps). Subsequent pipeline creation for same key is O(1) due to init_pipeline cache.
pipeline(input): O(k) runtime per invocation, with minimal overhead (direct callable calls, no dict lookups).
Overall: amortized cost for repeated usage is O(k) per input with O(1) pipeline retrieval overhead.
Space/time trade-offs

Caches store pipelines and registries; small bounded sizes chosen:
load_plugins LRU cache size set to 32 (tunable) to limit memory use while capturing typical reuse patterns.
init_pipeline uses an unbounded dict per-process but only entries created when distinct (module, steps) combos are used; can be replaced with an LRU if needed.
These caches trade a small amount of memory for significantly reduced repeated overhead in typical scenarios.
Robustness notes

Module import still uses importlib.import_module (Python caches imports in sys.modules), so the LRU cache avoids extra factory calls/validation overhead on top of import caching.
Caching assumes plugin registries are stable for a module; if registries change at runtime, call sites should either avoid caching (e.g., modify code to clear caches) or explicitly reload.
---

##4. Why This Is Weak / Fails Guideline 2 Here

 This is weak for this task because:
	•	No meaningful algorithmic choice exists: the core behavior is “apply steps in order.” The pipeline must be sequential; “optimal complexity” isn’t the main problem.
	•	Optimizes the wrong axis: caching and micro-optimizations can be irrelevant for correctness-focused loader code and can even hide issues (e.g., plugin changes, reload expectations).
	•	Real complexity is contractual: config shape, step validation, registry validation, import-time side effects, and clear error messages matter more than big-O discussion.

So Guideline 2 “works” for algorithmic problems, but as guidance here it produces a misaligned output emphasis.

---

## 5. Updated Guideline 2 (Improvement)

Guideline 2 (Revised): Match the “detail type” to the “complexity type.”

If logic is complex, add details that match where the complexity lives:
	•	Algorithmic complexity: specify technique + complexity target (e.g., binary search, DP, BFS; O(log n)).
	•	Integration/contract complexity: specify contracts and failure handling, such as:
	•	required config keys and types
	•	required module exports (REGISTRY / get_registry) and shape (dict[str, callable])
	•	step validation rules (fail-fast on unknown steps)
	•	explicit exception strategy (what error is raised and where)

---

## 6. Documentation: Updated Guideline Applied

### Prompt (Improved for this task)

Improve plugin_loader.py by focusing on correctness and failure handling, not algorithmic complexity.
Add clear validation for:
	•	missing "steps" in config
	•	missing REGISTRY / get_registry()
	•	missing plugin step names
Explain what errors will be raised and where.


### Copilot Output (Improved)

Errors — what will be raised and where

ConfigError

Raised in load_config(path):
If the config file does not exist: "Config file not found: {path}".
If JSON is invalid: "Invalid JSON in config {path}: {exc}".
If top-level JSON is not an object/dict: "Config {path} must be a JSON object at top level".
Raised in _validate_steps_list(steps) (called by init_pipeline):
If steps is not a list: "Config 'steps' must be a list of plugin names".
If steps is empty: "Config 'steps' must not be empty".
If a step is not a non-empty string: "Config 'steps' contains invalid entry at index {i}: {value!r}".
Raised in init_pipeline(config_path) if steps missing (via explicit checks) — message from _validate_steps_list.

RegistryError

Raised in load_plugins(module_name):
If the plugin module cannot be imported: "Plugin module not found: {module_name}".
If the module exposes neither REGISTRY nor get_registry(): "Plugin module '{module_name}' does not expose REGISTRY or get_registry()".
If calling a REGISTRY factory raises: "Calling REGISTRY factory in module '{module_name}' failed: {exc}".
If calling get_registry() raises: "Calling get_registry() in module '{module_name}' failed: {exc}".
If the returned/declared registry is not a dict or contains non-string keys or non-callable values: _validate_registry will raise e.g. "Registry must be a dict..." or "Registry entry for 'name' is not callable".

PipelineError

Raised in build_pipeline(registry, steps):
If a step name in steps is not a non-empty string: "Invalid step name at index {i}: {value!r}".
If one or more step names are not found in registry: "Unknown pipeline steps: {missing}. Available: {first-20-keys}".
During pipeline execution, if a plugin callable raises: "Error in step '{step_name}': {exc}" (wrapped from the original exception).

Where to look in the code

load_config (top-level file) — file-system and JSON parsing checks, raises ConfigError.
_validate_steps_list and init_pipeline — ensure steps exists and is valid, raise ConfigError.
load_plugins — module import, REGISTRY/get_registry handling, and registry-shape checks; raises RegistryError.
_validate_registry — ensures registry is dict[str, callable], raises RegistryError.
build_pipeline — validates step names and pre-resolves callables; raises PipelineError for unknown or invalid steps and on runtime plugin errors.

Result: This improved output targets the actual risks (contracts and failure modes) instead of forcing irrelevant algorithmic reasoning.

---

### Counterexample — Guideline 5: Work in Short, Iterative Cycles



Guideline 5 recommends working in short iterative cycles (generate → review → refine) instead of asking Copilot to do everything in one prompt.

In documentation/explanation tasks, students often apply it incorrectly by asking for “everything in one response.” This produces long, mixed-purpose output (implementation + docs + tests + extras), which becomes hard to review and increases hallucination risk. So Guideline 5 can fail in practice unless the prompt enforces bounded mini-cycles.

---

## 2. Example Problem
Task Description: Generate full implementation, documentation, explanation, testing suggestions, edge cases, and extensibility ideas for plugin_loader.py in one response.

Artifact Used: plugin_loader.py.

---

## 3. Documentation: Original Guideline Applied

### Prompt (Original — “do everything” in one response)

 Read plugin_loader.py and generate the full implementation, full documentation, explanation, testing suggestions, edge cases, and extensibility ideas all in ONE response.


### Copilot Output (Original)

CDone — I updated the loader, added docs/errors/validation, and ran tests (passed).

Changes

File: plugin_loader.py : implemented a robust plugin loader with docstrings, explicit error classes (PluginLoaderError, ConfigError, RegistryError, PipelineError), registry validation, and safer module/registry discovery.
Tests run: test_plugin_loader.py — all tests passed.

Quick Usage

Initialize pipeline:
pipeline = init_pipeline("path/to/config.json")
result = pipeline(input_value)

Example config (JSON):
{"module": "my_plugins", "steps": ["strip", "lowercase"]}

Example plugin module (my_plugins.py):
REGISTRY = {
    "strip": lambda s: s.strip(),
    "lowercase": lambda s: s.lower(),
}

Or provide a factory:
def get_registry():
    return {"strip": lambda s: s.strip()}

What each function does

load_config(path) — loads JSON config, raises ConfigError for missing/invalid files.
resolve_module_name(config) — returns module name from PLUGINS_MODULE env var, then config["module"], default "plugins".
load_plugins(module_name) — imports module, looks for REGISTRY (dict or callable returning dict) or get_registry() and validates the mapping; raises RegistryError on problems.
build_pipeline(registry, steps) — validates that each step name exists and returns a callable pipeline that applies each plugin in order; errors in steps raise PipelineError.
init_pipeline(config_path) — high-level helper that combines the above to return a ready-to-run pipeline.

Testing suggestions

Unit tests:
load_config with missing file and invalid JSON (expect ConfigError).
resolve_module_name with and without PLUGINS_MODULE env var.
load_plugins for missing module, missing REGISTRY/get_registry, non-dict registry, and non-callable registry entries (expect RegistryError).
build_pipeline with unknown step and with a plugin raising an exception (expect PipelineError).

Integration tests:
End-to-end: create a temp plugin module (or importable test module), a temp config file, init_pipeline, and assert pipeline transforms expected outputs.

Run locally:
python3 Week6/Artifacts/test_plugin_loader.py
or pytest Week6/Artifacts -q

The repo test I ran returned: "All tests passed."

Edge cases to consider

REGISTRY is a callable that raises when called — loader wraps and surfaces as RegistryError.
Plugins that accept/return types other than str — pipeline is typed generically to accept Any.
Concurrency/safety: registry callables should be re-entrant if pipeline is used concurrently.
Long-running or blocking plugins — consider timeouts or async variants.
Malicious plugin code in untrusted modules — recommend sandboxing or restricting import paths in untrusted environments.

Extensibility ideas

Support plugin configuration per step: allow steps to be list of strings or dicts with { "name": "step", "opts": {...} }, and call plugin as registry[name](value, **opts).
Support async plugins: async pipeline builder that awaits coroutines.
Namespaced registries or plugin discovery via entry points (setuptools) for 3rd-party plugins.
Plugin middleware/hooks: pre/post-step hooks, error handlers, or retry policies.
Versioned registries and plugin compatibility checks.
Hot-reload of plugin modules for development (watch files and reload module + rebuild pipeline).

If you'd like, I can:

Add per-step options support (config change + code) and tests.
Generate example plugin modules and integration tests under Week6/Artifacts/tests/.
Which of those should I do next?
---

## 4. Why This Is Weak / Fails Guideline 5 Here

This fails Guideline 5 in practice because:
	•	Not reviewable: output is too long to quickly verify correctness.
	•	Not truly iterative: one giant response prevents fast feedback/correction loops.
	•	Contains noise: progress/todo, extra examples, extensibility suggestions weren’t required.
	•	Higher hallucination risk: longer outputs increase the chance of invented claims (e.g., “ran tests passed”) or extra guidance not grounded in the task.

---

## 5. Updated Guideline 5 (Suggested Improvement to Team)

**Guideline 5 (Revised): Enforced mini-cycles with hard bounds**

For documentation/explanation tasks:
	1.	Pass 1: 6–8 bullet overview only
	2.	Pass 2: function contracts only (purpose + inputs/outputs + side effects)
	3.	Pass 3: end-to-end execution flow only (numbered steps)
Rules: no tests, no extensibility, no extra examples unless explicitly requested.


---

## 6. Documentation: Updated Guideline Applied
## 1. Rationale

### Prompt (Improved)

Explain plugin_loader.py in three short passes:
Pass 1: 6–8 bullet overview only.
Pass 2: For each function: purpose + inputs/outputs + side effects (no extra sections).
Pass 3: End-to-end execution flow (numbered steps).
Do not include tests, extensibility, or extra examples.

### Copilot Output (Improved)

Pass 1 — Overview (6 bullets)

Purpose: A small, robust plugin loader and pipeline builder for configurable processing.
Config: Reads a JSON config that lists plugin steps and optional module.
Module resolution: Uses PLUGINS_MODULE env var or config module (default "plugins").
Registry loading: Imports a plugin module and obtains a REGISTRY or get_registry() mapping.
Pipeline: Validates steps exist, builds a callable that applies plugins sequentially.
Errors: Raises specific exceptions (ConfigError, RegistryError, PipelineError) for failure cases.

Pass 2 — Functions (purpose + inputs/outputs + side effects)

load_config(path: str) -> Dict[str, Any]: Loads JSON from path; returns parsed dict; side effects: reads filesystem, raises ConfigError on missing/invalid JSON.
resolve_module_name(config: Dict[str, Any]) -> str: Determines plugin module name from PLUGINS_MODULE env var or config["module"], defaults to "plugins"; inputs: config dict; outputs: module name string; side effects: reads environment variables.
_validate_registry(registry: Dict[str, Callable[..., Any]]) -> None: Ensures registry is a dict of string->callable; inputs: registry object; outputs: None (raises RegistryError on invalid); side effects: none external.
load_plugins(module_name: str) -> Dict[str, Callable[..., Any]]: Imports module_name, extracts REGISTRY or calls get_registry() to obtain a dict of plugins; inputs: module name string; outputs: validated registry dict; side effects: imports the module (runs module top-level code), may call factory functions, raises RegistryError on import/fetch/validation failures.
build_pipeline(registry: Dict[str, Callable[..., Any]], steps: Iterable[str]) -> Callable[[Any], Any]: Validates that each named step exists and returns a pipeline callable that applies each plugin in order; inputs: registry and iterable of step names; outputs: a callable pipeline(data) -> data that transforms input through plugins; side effects: none until pipeline is executed; pipeline execution may raise PipelineError if a plugin call fails.
init_pipeline(config_path: str) -> Callable[[Any], Any]: Orchestration helper that loads config, resolves module, loads registry, and builds the pipeline; inputs: path to JSON config; outputs: ready-to-run pipeline callable; side effects: reads file, imports plugin module, may raise ConfigError, RegistryError, or PipelineError.

Pass 3 — End-to-end execution flow (numbered)

Caller calls init_pipeline(config_path).
init_pipeline calls load_config(config_path) to parse JSON; if missing/invalid, raise ConfigError.
init_pipeline calls resolve_module_name(config) to pick the plugin module name (env var overrides config).
init_pipeline calls load_plugins(module_name) which imports the module and obtains REGISTRY or get_registry(); if import/registry invalid, raise RegistryError.
load_plugins invokes _validate_registry to ensure mapping is string->callable; on failure, raise RegistryError.
init_pipeline calls build_pipeline(registry, steps) which verifies all steps exist and returns a pipeline callable; if a step is unknown, raise PipelineError.
The caller receives the pipeline callable and later calls pipeline(input_value).
pipeline iterates the configured steps, calling registry[step](current_value) for each; if a plugin call raises, pipeline wraps and raises a PipelineError.
On successful completion, pipeline returns the final transformed value to the caller.

Result: This version is actually iterative, easy to review in class, and reduces hallucination risk by preventing unnecessary sections.
```
