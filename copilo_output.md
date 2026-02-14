Prompt (Original / fails Guideline 5)
Read plugin_loader.py and generate the full implementation, full documentation, explanation, testing suggestions, edge cases, and extensibility ideas all in ONE response.

GITHUB COPILOT OUTPUT:
Full Implementation
import importlib
import functools
import json
import os
from pathlib import Path
from typing import Callable, Dict, Any, Iterable, List, Tuple

class PluginLoaderError(Exception):
"""Base exception for plugin loader errors."""

class ConfigError(PluginLoaderError):
"""Raised when configuration is missing or invalid."""

class RegistryError(PluginLoaderError):
"""Raised when plugin registry cannot be loaded or is invalid."""

class PipelineError(PluginLoaderError):
"""Raised when pipeline construction or execution fails."""

def load_config(path: str) -> Dict[str, Any]:
"""Load a JSON configuration file and return it as a dictionary.

    Args:
        path: Path to the JSON configuration file.

    Returns:
        The parsed configuration dictionary.

    Raises:
        ConfigError: If the file can't be read or JSON is invalid.
    """
    p = Path(path)
    if not p.exists():
        raise ConfigError(f"Config file not found: {path}")
    try:
        with p.open("r", encoding="utf-8") as f:
            cfg = json.load(f)
    except json.JSONDecodeError as exc:
        raise ConfigError(f"Invalid JSON in config {path}: {exc}") from exc

    if not isinstance(cfg, dict):
        raise ConfigError(f"Config {path} must be a JSON object at top level")

    return cfg

def resolve_module_name(config: Dict[str, Any]) -> str:
"""Resolve the plugins module name.

    Priority: environment variable `PLUGINS_MODULE` overrides config key `module`.
    Defaults to "plugins" when neither is provided.
    """
    env = os.getenv("PLUGINS_MODULE")
    if env:
        return env
    module_name = config.get("module") if isinstance(config, dict) else None
    return module_name or "plugins"

def \_validate_registry(registry: Dict[str, Callable[..., Any]]) -> None:
if not isinstance(registry, dict):
raise RegistryError("Registry must be a dict of name->callable")
for name, fn in registry.items():
if not isinstance(name, str):
raise RegistryError(f"Registry key is not str: {name!r}")
if not callable(fn):
raise RegistryError(f"Registry entry for '{name}' is not callable")

def \_validate_steps_list(steps: Any) -> List[str]:
"""Validate that `steps` is a non-empty list of non-empty strings.

    Returns the validated list of step names.
    Raises ConfigError on invalid input.
    """
    if not isinstance(steps, list):
        raise ConfigError("Config 'steps' must be a list of plugin names")
    if not steps:
        raise ConfigError("Config 'steps' must not be empty")
    validated: List[str] = []
    for i, s in enumerate(steps):
        if not isinstance(s, str) or not s:
            raise ConfigError(f"Config 'steps' contains invalid entry at index {i}: {s!r}")
        validated.append(s)
    return validated

@functools.lru_cache(maxsize=32)
def load_plugins(module_name: str) -> Dict[str, Callable[..., Any]]:
"""Import a module and return a plugins registry.

    The loader looks for the following in order on the imported module:
    - a top-level `REGISTRY` object (dict or callable returning dict)
    - a callable `get_registry()` which returns the dict

    Raises RegistryError when the module cannot be imported or the
    registry doesn't conform to the expected shape.
    """
    try:
        module = importlib.import_module(module_name)
    except ModuleNotFoundError as exc:
        raise RegistryError(f"Plugin module not found: {module_name}") from exc

    # Prefer an explicit REGISTRY attribute. It may be a dict or a callable
    # factory returning a dict.
    if hasattr(module, "REGISTRY"):
        reg = getattr(module, "REGISTRY")
        if callable(reg):
            try:
                reg = reg()
            except Exception as exc:
                raise RegistryError(f"Calling REGISTRY factory in module '{module_name}' failed: {exc}") from exc
    elif hasattr(module, "get_registry") and callable(getattr(module, "get_registry")):
        try:
            reg = getattr(module, "get_registry")()
        except Exception as exc:
            raise RegistryError(f"Calling get_registry() in module '{module_name}' failed: {exc}") from exc
    else:
        raise RegistryError(f"Plugin module '{module_name}' does not expose REGISTRY or get_registry()")

    _validate_registry(reg)
    return reg

def build_pipeline(registry: Dict[str, Callable[..., Any]], steps: Iterable[str]) -> Callable[[Any], Any]:
"""Build a pipeline function that applies named plugins in order.

    Args:
        registry: A mapping of plugin name -> callable.
        steps: An iterable of plugin names to apply in order.

    Returns:
        A callable that accepts one argument (input) and returns the transformed output.

    Raises:
        PipelineError: If any step name is missing from the registry.
    """
    step_list: List[str] = list(steps)

    # Ensure step names are strings
    for i, s in enumerate(step_list):
        if not isinstance(s, str) or not s:
            raise PipelineError(f"Invalid step name at index {i}: {s!r}")

    # Validate steps exist in registry. Using local vars to reduce attribute lookups.
    missing = [s for s in step_list if s not in registry]
    if missing:
        available = list(registry.keys())[:20]
        raise PipelineError(f"Unknown pipeline steps: {missing}. Available: {available}")

    # Pre-resolve callables to avoid dict lookups during pipeline execution.
    callables: List[Callable[[Any], Any]] = [registry[s] for s in step_list]

    def pipeline(data: Any) -> Any:
        value = data
        for idx, fn in enumerate(callables):
            try:
                value = fn(value)
            except Exception as exc:
                # Provide the step name in the error using step_list
                raise PipelineError(f"Error in step '{step_list[idx]}': {exc}") from exc
        return value

    return pipeline

def init_pipeline(config_path: str) -> Callable[[Any], Any]:
"""Initialize a processing pipeline from a JSON configuration file.

    Config format example:
    {
        "module": "my_plugins",
        "steps": ["strip", "lowercase", "tokenize"]
    }

    The `module` key is optional and can be overridden by the
    `PLUGINS_MODULE` environment variable.
    """
    config = load_config(config_path)
    if "steps" not in config:
        raise ConfigError("Config must include a top-level 'steps' list")
    steps = _validate_steps_list(config["steps"])

    module_name = resolve_module_name(config)
    registry = load_plugins(module_name)

    # Cache built pipelines keyed by (module_name, tuple(steps)). This avoids
    # rebuilding the same pipeline repeatedly when init_pipeline is called
    # multiple times with identical configs. We keep a small in-module cache
    # as the registry objects are typically stable for a module.
    if not hasattr(init_pipeline, "_pipeline_cache"):
        init_pipeline._pipeline_cache = {}  # type: ignore[attr-defined]

    key: Tuple[str, Tuple[str, ...]] = (module_name, tuple(steps))
    cache = init_pipeline._pipeline_cache  # type: ignore[attr-defined]
    if key in cache:
        return cache[key]

    pipeline = build_pipeline(registry, steps)
    cache[key] = pipeline
    return pipeline

**all** = [
"load_config",
"resolve_module_name",
"load_plugins",
"build_pipeline",
"init_pipeline",
"PluginLoaderError",
"ConfigError",
"RegistryError",
"PipelineError",
]
import importlib
import functools
import json
import os
from pathlib import Path
from typing import Callable, Dict, Any, Iterable, List, Tuple

class PluginLoaderError(Exception):
"""Base exception for plugin loader errors."""

class ConfigError(PluginLoaderError):
"""Raised when configuration is missing or invalid."""

class RegistryError(PluginLoaderError):
"""Raised when plugin registry cannot be loaded or is invalid."""

class PipelineError(PluginLoaderError):
"""Raised when pipeline construction or execution fails."""

def load_config(path: str) -> Dict[str, Any]:
"""Load a JSON configuration file and return it as a dictionary.

    Args:
        path: Path to the JSON configuration file.

    Returns:
        The parsed configuration dictionary.

    Raises:
        ConfigError: If the file can't be read or JSON is invalid.
    """
    p = Path(path)
    if not p.exists():
        raise ConfigError(f"Config file not found: {path}")
    try:
        with p.open("r", encoding="utf-8") as f:
            cfg = json.load(f)
    except json.JSONDecodeError as exc:
        raise ConfigError(f"Invalid JSON in config {path}: {exc}") from exc

    if not isinstance(cfg, dict):
        raise ConfigError(f"Config {path} must be a JSON object at top level")

    return cfg

def resolve_module_name(config: Dict[str, Any]) -> str:
"""Resolve the plugins module name.

    Priority: environment variable `PLUGINS_MODULE` overrides config key `module`.
    Defaults to "plugins" when neither is provided.
    """
    env = os.getenv("PLUGINS_MODULE")
    if env:
        return env
    module_name = config.get("module") if isinstance(config, dict) else None
    return module_name or "plugins"

def \_validate_registry(registry: Dict[str, Callable[..., Any]]) -> None:
if not isinstance(registry, dict):
raise RegistryError("Registry must be a dict of name->callable")
for name, fn in registry.items():
if not isinstance(name, str):
raise RegistryError(f"Registry key is not str: {name!r}")
if not callable(fn):
raise RegistryError(f"Registry entry for '{name}' is not callable")

def \_validate_steps_list(steps: Any) -> List[str]:
"""Validate that `steps` is a non-empty list of non-empty strings.

    Returns the validated list of step names.
    Raises ConfigError on invalid input.
    """
    if not isinstance(steps, list):
        raise ConfigError("Config 'steps' must be a list of plugin names")
    if not steps:
        raise ConfigError("Config 'steps' must not be empty")
    validated: List[str] = []
    for i, s in enumerate(steps):
        if not isinstance(s, str) or not s:
            raise ConfigError(f"Config 'steps' contains invalid entry at index {i}: {s!r}")
        validated.append(s)
    return validated

@functools.lru_cache(maxsize=32)
def load_plugins(module_name: str) -> Dict[str, Callable[..., Any]]:
"""Import a module and return a plugins registry.

    The loader looks for the following in order on the imported module:
    - a top-level `REGISTRY` object (dict or callable returning dict)
    - a callable `get_registry()` which returns the dict

    Raises RegistryError when the module cannot be imported or the
    registry doesn't conform to the expected shape.
    """
    try:
        module = importlib.import_module(module_name)
    except ModuleNotFoundError as exc:
        raise RegistryError(f"Plugin module not found: {module_name}") from exc

    # Prefer an explicit REGISTRY attribute. It may be a dict or a callable
    # factory returning a dict.
    if hasattr(module, "REGISTRY"):
        reg = getattr(module, "REGISTRY")
        if callable(reg):
            try:
                reg = reg()
            except Exception as exc:
                raise RegistryError(f"Calling REGISTRY factory in module '{module_name}' failed: {exc}") from exc
    elif hasattr(module, "get_registry") and callable(getattr(module, "get_registry")):
        try:
            reg = getattr(module, "get_registry")()
        except Exception as exc:
            raise RegistryError(f"Calling get_registry() in module '{module_name}' failed: {exc}") from exc
    else:
        raise RegistryError(f"Plugin module '{module_name}' does not expose REGISTRY or get_registry()")

    _validate_registry(reg)
    return reg

def build_pipeline(registry: Dict[str, Callable[..., Any]], steps: Iterable[str]) -> Callable[[Any], Any]:
"""Build a pipeline function that applies named plugins in order.

    Args:
        registry: A mapping of plugin name -> callable.
        steps: An iterable of plugin names to apply in order.

    Returns:
        A callable that accepts one argument (input) and returns the transformed output.

    Raises:
        PipelineError: If any step name is missing from the registry.
    """
    step_list: List[str] = list(steps)

    # Ensure step names are strings
    for i, s in enumerate(step_list):
        if not isinstance(s, str) or not s:
            raise PipelineError(f"Invalid step name at index {i}: {s!r}")

    # Validate steps exist in registry. Using local vars to reduce attribute lookups.
    missing = [s for s in step_list if s not in registry]
    if missing:
        available = list(registry.keys())[:20]
        raise PipelineError(f"Unknown pipeline steps: {missing}. Available: {available}")

    # Pre-resolve callables to avoid dict lookups during pipeline execution.
    callables: List[Callable[[Any], Any]] = [registry[s] for s in step_list]

    def pipeline(data: Any) -> Any:
        value = data
        for idx, fn in enumerate(callables):
            try:
                value = fn(value)
            except Exception as exc:
                # Provide the step name in the error using step_list
                raise PipelineError(f"Error in step '{step_list[idx]}': {exc}") from exc
        return value

    return pipeline

def init_pipeline(config_path: str) -> Callable[[Any], Any]:
"""Initialize a processing pipeline from a JSON configuration file.

    Config format example:
    {
        "module": "my_plugins",
        "steps": ["strip", "lowercase", "tokenize"]
    }

    The `module` key is optional and can be overridden by the
    `PLUGINS_MODULE` environment variable.
    """
    config = load_config(config_path)
    if "steps" not in config:
        raise ConfigError("Config must include a top-level 'steps' list")
    steps = _validate_steps_list(config["steps"])

    module_name = resolve_module_name(config)
    registry = load_plugins(module_name)

    # Cache built pipelines keyed by (module_name, tuple(steps)). This avoids
    # rebuilding the same pipeline repeatedly when init_pipeline is called
    # multiple times with identical configs. We keep a small in-module cache
    # as the registry objects are typically stable for a module.
    if not hasattr(init_pipeline, "_pipeline_cache"):
        init_pipeline._pipeline_cache = {}  # type: ignore[attr-defined]

    key: Tuple[str, Tuple[str, ...]] = (module_name, tuple(steps))
    cache = init_pipeline._pipeline_cache  # type: ignore[attr-defined]
    if key in cache:
        return cache[key]

    pipeline = build_pipeline(registry, steps)
    cache[key] = pipeline
    return pipeline

**all** = [
"load_config",
"resolve_module_name",
"load_plugins",
"build_pipeline",
"init_pipeline",
"PluginLoaderError",
"ConfigError",
"RegistryError",
"PipelineError",
]
Full Documentation
The code includes comprehensive docstrings for all public functions and classes. Key documentation points:

Classes: PluginLoaderError and subclasses (ConfigError, RegistryError, PipelineError) provide specific exception types for different failure modes.
Functions: Each has detailed docstrings explaining purpose, arguments, returns, and raises.
Module-level: **all** lists exported symbols for clean imports.
Explanation
This is a robust plugin loader and pipeline builder for Python. It loads a JSON config specifying a plugin module and a sequence of steps (plugin names). It imports the module, extracts a registry of callable plugins, validates the steps, and builds a pipeline function that applies the plugins in order. Caching minimizes redundant work for repeated loads. Errors are handled with specific exceptions for config, registry, and pipeline issues.

Testing Suggestions
Unit Tests: Test each function in isolation (e.g., load_config with invalid JSON, load_plugins with missing module, build_pipeline with unknown steps).
Integration Tests: End-to-end tests with temp config files and modules, asserting pipeline output.
Error Tests: Verify specific exceptions and messages for invalid inputs.
Run Tests: Use python3 test_plugin_loader.py or a framework like pytest.
Edge Cases
Empty or invalid config (e.g., non-dict JSON, missing "steps").
Plugin module not found or lacking REGISTRY/get_registry().
Registry with non-callable values or invalid keys.
Steps with non-string names or unknown plugins.
Plugins that raise exceptions during execution.
Concurrent access (caching assumes stable registries).
Extensibility Ideas
Add per-step options (e.g., steps as dicts with args).
Support async plugins and pipelines.
Plugin discovery via entry points.
Middleware for hooks, retries, or logging.
Versioning and compatibility checks.
Hot-reload for development.

Improve Guideline 5 (force real mini-cycles)
Explain plugin_loader.py in three short passes:
Pass 1: 6–8 bullet overview only.
Pass 2: For each function: purpose + inputs/outputs + side effects (no extra sections).
Pass 3: End-to-end execution flow (numbered steps).
Do not include tests, extensibility, or extra examples.

GITHUB COPILOT OUTPUT:
Pass 1: 6–8 bullet overview only.

Purpose: A robust plugin loader and pipeline builder for configurable processing.
Config: Loads JSON config with plugin module and steps list.
Module resolution: Uses env var or config for module name, defaults to "plugins".
Registry loading: Imports module, extracts REGISTRY or get_registry(), validates as dict of callables.
Pipeline building: Validates steps exist, pre-resolves callables, returns a sequential application function.
Caching: LRU cache for registries, in-function cache for pipelines to avoid redundant work.
Errors: Specific exceptions (ConfigError, RegistryError, PipelineError) for validation failures.
Pass 2: For each function: purpose + inputs/outputs + side effects (no extra sections).

load_config(path: str) -> Dict[str, Any]: Loads and parses JSON config file; returns dict; side effects: reads filesystem, raises ConfigError on missing/invalid file or non-dict JSON.
resolve_module_name(config: Dict[str, Any]) -> str: Determines module name from env var or config; returns string; side effects: reads environment variables.
\_validate_registry(registry: Dict[str, Callable[..., Any]]) -> None: Validates registry structure; returns None; side effects: raises RegistryError on invalid dict or entries.
\_validate_steps_list(steps: Any) -> List[str]: Validates steps as non-empty list of strings; returns list; side effects: raises ConfigError on invalid input.
load_plugins(module_name: str) -> Dict[str, Callable[..., Any]]: Imports module and extracts registry; returns dict; side effects: imports module (runs top-level code), calls factories, raises RegistryError on import/registry issues, caches results.
build_pipeline(registry: Dict[str, Callable[..., Any]], steps: Iterable[str]) -> Callable[[Any], Any]: Builds pipeline callable; returns function; side effects: validates steps, raises PipelineError on unknown/invalid steps, pipeline execution may raise PipelineError on plugin errors.
init_pipeline(config_path: str) -> Callable[[Any], Any]: Orchestrates loading and building; returns pipeline; side effects: reads file, imports module, caches pipelines, raises ConfigError/RegistryError/PipelineError.
Pass 3: End-to-end execution flow (numbered steps).

Caller invokes init_pipeline(config_path).
init_pipeline calls load_config to parse JSON; raises ConfigError if file missing/invalid or not a dict.
init_pipeline validates "steps" via \_validate_steps_list; raises ConfigError if missing/empty/invalid.
init_pipeline calls resolve_module_name to get module name from env or config.
init_pipeline calls load_plugins (cached) to import module and get registry; raises RegistryError if module/registry invalid.
load_plugins validates registry via \_validate_registry; raises RegistryError on failure.
init_pipeline checks pipeline cache; if hit, returns cached pipeline.
Otherwise, init_pipeline calls build_pipeline to validate steps and build callable; raises PipelineError on unknown steps.
build_pipeline pre-resolves callables and returns pipeline function.
init_pipeline caches and returns the pipeline.
Caller calls pipeline(input); pipeline applies each callable sequentially; raises PipelineError on plugin errors.
Pipeline returns final transformed output.

Prompt (Original / fails Guideline 2)
Implement plugin_loader.py using an efficient algorithmic approach. Ensure optimal time complexity. Avoid redundant operations. Explain algorithmic design decisions clearly.
GITHUB COPILOT OUTPUT:
Full Implementation
import importlib
import functools
import json
import os
from pathlib import Path
from typing import Callable, Dict, Any, Iterable, List, Tuple

class PluginLoaderError(Exception):
"""Base exception for plugin loader errors."""

class ConfigError(PluginLoaderError):
"""Raised when configuration is missing or invalid."""

class RegistryError(PluginLoaderError):
"""Raised when plugin registry cannot be loaded or is invalid."""

class PipelineError(PluginLoaderError):
"""Raised when pipeline construction or execution fails."""

def load_config(path: str) -> Dict[str, Any]:
"""Load a JSON configuration file and return it as a dictionary.

    Args:
        path: Path to the JSON configuration file.

    Returns:
        The parsed configuration dictionary.

    Raises:
        ConfigError: If the file can't be read or JSON is invalid.
    """
    p = Path(path)
    if not p.exists():
        raise ConfigError(f"Config file not found: {path}")
    try:
        with p.open("r", encoding="utf-8") as f:
            cfg = json.load(f)
    except json.JSONDecodeError as exc:
        raise ConfigError(f"Invalid JSON in config {path}: {exc}") from exc

    if not isinstance(cfg, dict):
        raise ConfigError(f"Config {path} must be a JSON object at top level")

    return cfg

def resolve_module_name(config: Dict[str, Any]) -> str:
"""Resolve the plugins module name.

    Priority: environment variable `PLUGINS_MODULE` overrides config key `module`.
    Defaults to "plugins" when neither is provided.
    """
    env = os.getenv("PLUGINS_MODULE")
    if env:
        return env
    module_name = config.get("module") if isinstance(config, dict) else None
    return module_name or "plugins"

def \_validate_registry(registry: Dict[str, Callable[..., Any]]) -> None:
if not isinstance(registry, dict):
raise RegistryError("Registry must be a dict of name->callable")
for name, fn in registry.items():
if not isinstance(name, str):
raise RegistryError(f"Registry key is not str: {name!r}")
if not callable(fn):
raise RegistryError(f"Registry entry for '{name}' is not callable")

def \_validate_steps_list(steps: Any) -> List[str]:
"""Validate that `steps` is a non-empty list of non-empty strings.

    Returns the validated list of step names.
    Raises ConfigError on invalid input.
    """
    if not isinstance(steps, list):
        raise ConfigError("Config 'steps' must be a list of plugin names")
    if not steps:
        raise ConfigError("Config 'steps' must not be empty")
    validated: List[str] = []
    for i, s in enumerate(steps):
        if not isinstance(s, str) or not s:
            raise ConfigError(f"Config 'steps' contains invalid entry at index {i}: {s!r}")
        validated.append(s)
    return validated

@functools.lru_cache(maxsize=32)
def load_plugins(module_name: str) -> Dict[str, Callable[..., Any]]:
"""Import a module and return a plugins registry.

    The loader looks for the following in order on the imported module:
    - a top-level `REGISTRY` object (dict or callable returning dict)
    - a callable `get_registry()` which returns the dict

    Raises RegistryError when the module cannot be imported or the
    registry doesn't conform to the expected shape.
    """
    try:
        module = importlib.import_module(module_name)
    except ModuleNotFoundError as exc:
        raise RegistryError(f"Plugin module not found: {module_name}") from exc

    # Prefer an explicit REGISTRY attribute. It may be a dict or a callable
    # factory returning a dict.
    if hasattr(module, "REGISTRY"):
        reg = getattr(module, "REGISTRY")
        if callable(reg):
            try:
                reg = reg()
            except Exception as exc:
                raise RegistryError(f"Calling REGISTRY factory in module '{module_name}' failed: {exc}") from exc
    elif hasattr(module, "get_registry") and callable(getattr(module, "get_registry")):
        try:
            reg = getattr(module, "get_registry")()
        except Exception as exc:
            raise RegistryError(f"Calling get_registry() in module '{module_name}' failed: {exc}") from exc
    else:
        raise RegistryError(f"Plugin module '{module_name}' does not expose REGISTRY or get_registry()")

    _validate_registry(reg)
    return reg

def build_pipeline(registry: Dict[str, Callable[..., Any]], steps: Iterable[str]) -> Callable[[Any], Any]:
"""Build a pipeline function that applies named plugins in order.

    Args:
        registry: A mapping of plugin name -> callable.
        steps: An iterable of plugin names to apply in order.

    Returns:
        A callable that accepts one argument (input) and returns the transformed output.

    Raises:
        PipelineError: If any step name is missing from the registry.
    """
    step_list: List[str] = list(steps)

    # Ensure step names are strings
    for i, s in enumerate(step_list):
        if not isinstance(s, str) or not s:
            raise PipelineError(f"Invalid step name at index {i}: {s!r}")

    # Validate steps exist in registry. Using local vars to reduce attribute lookups.
    missing = [s for s in step_list if s not in registry]
    if missing:
        available = list(registry.keys())[:20]
        raise PipelineError(f"Unknown pipeline steps: {missing}. Available: {available}")

    # Pre-resolve callables to avoid dict lookups during pipeline execution.
    callables: List[Callable[[Any], Any]] = [registry[s] for s in step_list]

    def pipeline(data: Any) -> Any:
        value = data
        for idx, fn in enumerate(callables):
            try:
                value = fn(value)
            except Exception as exc:
                # Provide the step name in the error using step_list
                raise PipelineError(f"Error in step '{step_list[idx]}': {exc}") from exc
        return value

    return pipeline

def init_pipeline(config_path: str) -> Callable[[Any], Any]:
"""Initialize a processing pipeline from a JSON configuration file.

    Config format example:
    {
        "module": "my_plugins",
        "steps": ["strip", "lowercase", "tokenize"]
    }

    The `module` key is optional and can be overridden by the
    `PLUGINS_MODULE` environment variable.
    """
    config = load_config(config_path)
    if "steps" not in config:
        raise ConfigError("Config must include a top-level 'steps' list")
    steps = _validate_steps_list(config["steps"])

    module_name = resolve_module_name(config)
    registry = load_plugins(module_name)

    # Cache built pipelines keyed by (module_name, tuple(steps)). This avoids
    # rebuilding the same pipeline repeatedly when init_pipeline is called
    # multiple times with identical configs. We keep a small in-module cache
    # as the registry objects are typically stable for a module.
    if not hasattr(init_pipeline, "_pipeline_cache"):
        init_pipeline._pipeline_cache = {}  # type: ignore[attr-defined]

    key: Tuple[str, Tuple[str, ...]] = (module_name, tuple(steps))
    cache = init_pipeline._pipeline_cache  # type: ignore[attr-defined]
    if key in cache:
        return cache[key]

    pipeline = build_pipeline(registry, steps)
    cache[key] = pipeline
    return pipeline

**all** = [
"load_config",
"resolve_module_name",
"load_plugins",
"build_pipeline",
"init_pipeline",
"PluginLoaderError",
"ConfigError",
"RegistryError",
"PipelineError",
]
Algorithmic Design Decisions
Key goals: Minimize redundant work in repeated usage (e.g., re-imports, re-validations, rebuilds), optimize per-input pipeline execution, maintain bounded caches for predictability.

Caching plugin registry: @functools.lru_cache(maxsize=32) on load_plugins caches validated registries by module name. First load: O(R) for validation (R = registry size); subsequent: O(1) amortized. Avoids re-importing and re-calling factories.
Pipeline construction cache: In init_pipeline, cache pipelines by (module_name, tuple(steps)). Lookup: O(1); first build: O(k + m) (k = steps, m = registry size for validation); subsequent: O(1). Prevents rebuilding identical pipelines.
Pre-resolving callables: In build_pipeline, create a list of callables upfront. Build: O(k); execution: O(k) with lower constant factor (direct calls vs. dict lookups).
Validation efficiency: Missing steps detected in one pass with list comprehension. Registry validation cached per module.
Time complexity summary: load_config: O(C) (C = file size); load_plugins: O(R) first, O(1) cached; build_pipeline: O(k) first, O(1) cached; pipeline(input): O(k) per input.
Space/time trade-offs: LRU cache (32 entries) and unbounded pipeline cache (per-process, grows with unique combos) trade memory for speed in repeated scenarios. Assumes stable registries; for dynamic changes, clear caches explicitly.

Improve Guideline 2 (make it “domain-relevant”)
Improve plugin_loader.py by focusing on correctness and failure handling, not algorithmic complexity.
Add clear validation for:
• missing "steps" in config
• missing REGISTRY
• missing plugin step names
Explain what errors will be raised and where.
GITHUB COPILOT OUTPUT:
Full Implementation
import importlib
import functools
import json
import os
from pathlib import Path
from typing import Callable, Dict, Any, Iterable, List, Tuple

class PluginLoaderError(Exception):
"""Base exception for plugin loader errors."""

class ConfigError(PluginLoaderError):
"""Raised when configuration is missing or invalid."""

class RegistryError(PluginLoaderError):
"""Raised when plugin registry cannot be loaded or is invalid."""

class PipelineError(PluginLoaderError):
"""Raised when pipeline construction or execution fails."""

def load_config(path: str) -> Dict[str, Any]:
"""Load a JSON configuration file and return it as a dictionary.

    Args:
        path: Path to the JSON configuration file.

    Returns:
        The parsed configuration dictionary.

    Raises:
        ConfigError: If the file can't be read or JSON is invalid.
    """
    p = Path(path)
    if not p.exists():
        raise ConfigError(f"Config file not found: {path}")
    try:
        with p.open("r", encoding="utf-8") as f:
            cfg = json.load(f)
    except json.JSONDecodeError as exc:
        raise ConfigError(f"Invalid JSON in config {path}: {exc}") from exc

    if not isinstance(cfg, dict):
        raise ConfigError(f"Config {path} must be a JSON object at top level")

    return cfg

def resolve_module_name(config: Dict[str, Any]) -> str:
"""Resolve the plugins module name.

    Priority: environment variable `PLUGINS_MODULE` overrides config key `module`.
    Defaults to "plugins" when neither is provided.
    """
    env = os.getenv("PLUGINS_MODULE")
    if env:
        return env
    module_name = config.get("module") if isinstance(config, dict) else None
    return module_name or "plugins"

def \_validate_registry(registry: Dict[str, Callable[..., Any]]) -> None:
if not isinstance(registry, dict):
raise RegistryError("Registry must be a dict of name->callable")
for name, fn in registry.items():
if not isinstance(name, str):
raise RegistryError(f"Registry key is not str: {name!r}")
if not callable(fn):
raise RegistryError(f"Registry entry for '{name}' is not callable")

def \_validate_steps_list(steps: Any) -> List[str]:
"""Validate that `steps` is a non-empty list of non-empty strings.

    Returns the validated list of step names.
    Raises ConfigError on invalid input.
    """
    if not isinstance(steps, list):
        raise ConfigError("Config 'steps' must be a list of plugin names")
    if not steps:
        raise ConfigError("Config 'steps' must not be empty")
    validated: List[str] = []
    for i, s in enumerate(steps):
        if not isinstance(s, str) or not s:
            raise ConfigError(f"Config 'steps' contains invalid entry at index {i}: {s!r}")
        validated.append(s)
    return validated

@functools.lru_cache(maxsize=32)
def load_plugins(module_name: str) -> Dict[str, Callable[..., Any]]:
"""Import a module and return a plugins registry.

    The loader looks for the following in order on the imported module:
    - a top-level `REGISTRY` object (dict or callable returning dict)
    - a callable `get_registry()` which returns the dict

    Raises RegistryError when the module cannot be imported or the
    registry doesn't conform to the expected shape.
    """
    try:
        module = importlib.import_module(module_name)
    except ModuleNotFoundError as exc:
        raise RegistryError(f"Plugin module not found: {module_name}") from exc

    # Prefer an explicit REGISTRY attribute. It may be a dict or a callable
    # factory returning a dict.
    if hasattr(module, "REGISTRY"):
        reg = getattr(module, "REGISTRY")
        if callable(reg):
            try:
                reg = reg()
            except Exception as exc:
                raise RegistryError(f"Calling REGISTRY factory in module '{module_name}' failed: {exc}") from exc
    elif hasattr(module, "get_registry") and callable(getattr(module, "get_registry")):
        try:
            reg = getattr(module, "get_registry")()
        except Exception as exc:
            raise RegistryError(f"Calling get_registry() in module '{module_name}' failed: {exc}") from exc
    else:
        raise RegistryError(f"Plugin module '{module_name}' does not expose REGISTRY or get_registry()")

    _validate_registry(reg)
    return reg

def build_pipeline(registry: Dict[str, Callable[..., Any]], steps: Iterable[str]) -> Callable[[Any], Any]:
"""Build a pipeline function that applies named plugins in order.

    Args:
        registry: A mapping of plugin name -> callable.
        steps: An iterable of plugin names to apply in order.

    Returns:
        A callable that accepts one argument (input) and returns the transformed output.

    Raises:
        PipelineError: If any step name is missing from the registry.
    """
    step_list: List[str] = list(steps)

    # Ensure step names are strings
    for i, s in enumerate(step_list):
        if not isinstance(s, str) or not s:
            raise PipelineError(f"Invalid step name at index {i}: {s!r}")

    # Validate steps exist in registry. Using local vars to reduce attribute lookups.
    missing = [s for s in step_list if s not in registry]
    if missing:
        available = list(registry.keys())[:20]
        raise PipelineError(f"Unknown pipeline steps: {missing}. Available: {available}")

    # Pre-resolve callables to avoid dict lookups during pipeline execution.
    callables: List[Callable[[Any], Any]] = [registry[s] for s in step_list]

    def pipeline(data: Any) -> Any:
        value = data
        for idx, fn in enumerate(callables):
            try:
                value = fn(value)
            except Exception as exc:
                # Provide the step name in the error using step_list
                raise PipelineError(f"Error in step '{step_list[idx]}': {exc}") from exc
        return value

    return pipeline

def init_pipeline(config_path: str) -> Callable[[Any], Any]:
"""Initialize a processing pipeline from a JSON configuration file.

    Config format example:
    {
        "module": "my_plugins",
        "steps": ["strip", "lowercase", "tokenize"]
    }

    The `module` key is optional and can be overridden by the
    `PLUGINS_MODULE` environment variable.
    """
    config = load_config(config_path)
    if "steps" not in config:
        raise ConfigError("Config must include a top-level 'steps' list")
    steps = _validate_steps_list(config["steps"])

    module_name = resolve_module_name(config)
    registry = load_plugins(module_name)

    # Cache built pipelines keyed by (module_name, tuple(steps)). This avoids
    # rebuilding the same pipeline repeatedly when init_pipeline is called
    # multiple times with identical configs. We keep a small in-module cache
    # as the registry objects are typically stable for a module.
    if not hasattr(init_pipeline, "_pipeline_cache"):
        init_pipeline._pipeline_cache = {}  # type: ignore[attr-defined]

    key: Tuple[str, Tuple[str, ...]] = (module_name, tuple(steps))
    cache = init_pipeline._pipeline_cache  # type: ignore[attr-defined]
    if key in cache:
        return cache[key]

    pipeline = build_pipeline(registry, steps)
    cache[key] = pipeline
    return pipeline

**all** = [
"load_config",
"resolve_module_name",
"load_plugins",
"build_pipeline",
"init_pipeline",
"PluginLoaderError",
"ConfigError",
"RegistryError",
"PipelineError",
]
Explanation of Errors Raised and Where
ConfigError:

Raised in load_config(path): If config file not found ("Config file not found: {path}"); if JSON invalid ("Invalid JSON in config {path}: {exc}"); if top-level not a dict ("Config {path} must be a JSON object at top level").
Raised in \_validate_steps_list(steps) (called by init_pipeline): If "steps" not a list ("Config 'steps' must be a list of plugin names"); if empty ("Config 'steps' must not be empty"); if entry not non-empty string ("Config 'steps' contains invalid entry at index {i}: {value!r}").
Raised in init_pipeline(config_path): If "steps" key missing ("Config must include a top-level 'steps' list").
RegistryError:

Raised in load_plugins(module_name): If module not found ("Plugin module not found: {module_name}"); if no REGISTRY/get_registry() ("Plugin module '{module_name}' does not expose REGISTRY or get_registry()"); if REGISTRY factory fails ("Calling REGISTRY factory in module '{module_name}' failed: {exc}"); if get_registry() fails ("Calling get_registry() in module '{module_name}' failed: {exc}").
Raised in \_validate_registry(registry) (called by load_plugins): If registry not dict ("Registry must be a dict of name->callable"); if key not string ("Registry key is not str: {name!r}"); if value not callable ("Registry entry for '{name}' is not callable").
PipelineError:

Raised in build_pipeline(registry, steps): If step name invalid ("Invalid step name at index {i}: {value!r}"); if step missing from registry ("Unknown pipeline steps: {missing}. Available: {available}").
Raised during pipeline execution (in returned callable): If plugin call fails ("Error in step '{step_name}': {exc}").
