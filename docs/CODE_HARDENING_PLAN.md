# Code Hardening and Cleanup Plan

**Date**: 2026-01-02  
**Status**: 🚧 In Progress  
**Target**: Improve code quality, maintainability, and testability

---

## Executive Summary

This document outlines a comprehensive plan to harden and clean up the `sdk-python` codebase following best practices for:
- **Code Reusability**: Reduce duplication through abstraction
- **Type Safety**: Replace `dict` return types with concrete Pydantic models
- **Modularity**: Improve code organization and separation of concerns
- **Testability**: Ensure core logic is easily testable with mocks
- **Code Quality**: Enforce linting and style consistency

---

## Principles

1. **Reduce Repeated Code**: Extract common patterns into reusable functions/classes
2. **Use Abstraction**: Create abstract base classes and interfaces where appropriate
3. **Modular Structure**: Organize code to minimize repetition and improve maintainability
4. **Testability**: Core logic should be testable with real instances or mocks (pytest)
5. **Prefer Concrete Class Types**: Use `@dataclass` and Pydantic `BaseModel` instead of `dict` return types
6. **Clean Code**: Code should be readable, well-documented, and follow Python best practices
7. **Code Linting**: Set up `pre-commit` hooks and GitHub Actions for automated linting

---

## Phase 1: Type Safety Improvements

### 1.0 Standardize Optional Type Hints

**Priority**: 🔴 High  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Current State

The codebase is inconsistent with optional type hints:
- **124 instances** of `str | None`, `int | None`, `dict | None`, etc. across 17 files
- Some places already use `Optional[str] = None` syntax (e.g., `agent.py`)
- Project requires Python 3.11+, so both work, but we want consistency

#### Standardization Decision

**Standardize on `Optional[str] = None` syntax** for consistency and explicit imports.

**Rationale:**
- More explicit about optionality
- Consistent with existing code in `agent.py`
- Clearer imports show dependencies
- Works well with forward references (`Optional["Tracer"]`)

#### Files to Update (17 files, 124 instances)

1. **`sentience/models.py`** - 24 instances
2. **`sentience/tracing.py`** - 10 instances
3. **`sentience/cloud_tracing.py`** - 5 instances
4. **`sentience/agent.py`** - 4 instances (already uses `Optional` in some places)
5. **`sentience/trace_indexing/index_schema.py`** - 14 instances
6. **`sentience/trace_indexing/indexer.py`** - 2 instances
7. **`sentience/tracer_factory.py`** - 3 instances
8. **`sentience/snapshot.py`** - 1 instance
9. **`sentience/screenshot.py`** - 2 instances
10. **`sentience/recorder.py`** - 13 instances
11. **`sentience/overlay.py`** - 2 instances
12. **`sentience/inspector.py`** - 2 instances
13. **`sentience/browser.py`** - 21 instances
14. **`sentience/base_agent.py`** - 2 instances
15. **`sentience/actions.py`** - 4 instances
16. **`sentience/llm_provider.py`** - 14 instances
17. **`sentience/utils.py`** - 1 instance

#### Implementation Steps

1. **Add imports**: Ensure `from typing import Optional` in all affected files

2. **Replace parameter type hints**:
   - `str | None = None` → `Optional[str] = None`
   - `int | None = None` → `Optional[int] = None`
   - `dict | None = None` → `Optional[dict] = None`
   - `list | None = None` → `Optional[list] = None`
   - `float | None = None` → `Optional[float] = None`
   - Similar patterns for other types

3. **Replace return type hints**:
   - `-> str | None` → `-> Optional[str]`
   - `-> int | None` → `-> Optional[int]`
   - `-> dict[str, Any] | None` → `-> Optional[dict[str, Any]]`
   - Similar patterns for other types

4. **Handle complex types**:
   - `dict[str, Any] | None` → `Optional[dict[str, Any]]`
   - `list[Element] | None` → `Optional[list[Element]]`
   - `Snapshot | None` → `Optional[Snapshot]`

5. **Keep forward references**: `Optional["Tracer"]` (quoted strings) is already correct

#### Example Transformations

```python
# Before
class VisualCues(BaseModel):
    background_color_name: str | None = None

class Element(BaseModel):
    text: str | None = None
    rerank_index: int | None = None

def get_stats(self) -> dict[str, Any] | None:
    return None

def _get_element_bbox(self, element_id: int | None, snap: Snapshot) -> dict[str, float] | None:
    return None

# After
from typing import Optional

class VisualCues(BaseModel):
    background_color_name: Optional[str] = None

class Element(BaseModel):
    text: Optional[str] = None
    rerank_index: Optional[int] = None

def get_stats(self) -> Optional[dict[str, Any]]:
    return None

def _get_element_bbox(self, element_id: Optional[int], snap: Snapshot) -> Optional[dict[str, float]]:
    return None
```

#### Automated Conversion Script

Create a script to automate the conversion:

```python
# scripts/convert_optional_types.py
import re
import sys
from pathlib import Path

def convert_file(file_path: Path) -> bool:
    """Convert | None to Optional[] in a file"""
    content = file_path.read_text(encoding='utf-8')
    original = content
    
    # Add Optional import if not present and file uses | None
    if '| None' in content and 'from typing import' in content:
        if 'Optional' not in content:
            # Add Optional to existing typing import
            content = re.sub(
                r'from typing import ([^#\n]+)',
                lambda m: f"from typing import {m.group(1)}, Optional" if 'Optional' not in m.group(1) else m.group(0),
                content,
                count=1
            )
    elif '| None' in content and 'from typing import' not in content:
        # Add new typing import at top of file
        lines = content.split('\n')
        import_line = 0
        for i, line in enumerate(lines):
            if line.startswith('import ') or line.startswith('from '):
                import_line = i + 1
        lines.insert(import_line, 'from typing import Optional')
        content = '\n'.join(lines)
    
    # Replace type hints
    patterns = [
        (r'(\w+)\s*\|\s*None\s*=', r'Optional[\1] ='),  # Parameter: str | None =
        (r'->\s*(\w+)\s*\|\s*None\s*:', r'-> Optional[\1]:'),  # Return: -> str | None:
        (r'(\w+\[[^\]]+\])\s*\|\s*None\s*=', r'Optional[\1] ='),  # Parameter: dict[str, Any] | None =
        (r'->\s*(\w+\[[^\]]+\])\s*\|\s*None\s*:', r'-> Optional[\1]:'),  # Return: -> dict[str, Any] | None:
    ]
    
    for pattern, replacement in patterns:
        content = re.sub(pattern, replacement, content)
    
    if content != original:
        file_path.write_text(content, encoding='utf-8')
        return True
    return False

if __name__ == '__main__':
    sentience_dir = Path('sentience')
    changed = 0
    for py_file in sentience_dir.rglob('*.py'):
        if convert_file(py_file):
            print(f"Converted: {py_file}")
            changed += 1
    print(f"\nConverted {changed} files")
```

#### Testing

- Run `mypy` to ensure type checking still works: `mypy sentience --ignore-missing-imports`
- Run existing tests: `pytest tests/`
- Verify imports are correct (no missing `Optional` imports)
- Check for any syntax errors: `python -m py_compile sentience/**/*.py`

#### Verification Checklist

- [x] All 124 instances converted
- [x] All files have `from typing import Optional` (or it's in existing import)
- [x] `mypy` passes with no new errors
- [x] All tests pass
- [x] No syntax errors introduced

---

### 1.1 Replace `dict` Return Types with Pydantic Models

**Priority**: 🔴 High  
**Estimated Effort**: 2-3 days  
**Status**: ✅ **Completed 2026-01-02**

#### Files to Update

1. **`sentience/read.py`** (Lines 10-14, 99-103)
   - **Current**: `read()` and `read_async()` return `dict`
   - **Target**: Create `ReadResult` Pydantic model
   ```python
   class ReadResult(BaseModel):
       status: Literal["success", "error"]
       url: str
       format: Literal["raw", "text", "markdown"]
       content: str
       length: int
       error: Optional[str] = None
   ```

2. **`sentience/tracing.py`** (Lines 33, 114, 434)
   - **Current**: `to_dict()` and `get_stats()` return `dict[str, Any]`
   - **Target**: Create concrete models:
     - `TraceStats` model for `get_stats()`
     - Keep `to_dict()` for serialization but add typed models

3. **`sentience/cloud_tracing.py`** (Lines 438, 584, 665)
   - **Current**: `_extract_stats_from_trace()` and `_extract_screenshots_from_trace()` return `dict`
   - **Target**: Create `TraceStats` and `ScreenshotMetadata` models

4. **`sentience/trace_indexing/indexer.py`** (Line 37)
   - **Current**: `_round_bbox()` returns `dict[str, int]`
   - **Target**: Use `BBox` model from `models.py`

5. **`sentience/conversational_agent.py`** (Lines 206, 306)
   - **Current**: `_execute_step()` and `_extract_information()` return `dict[str, Any]`
   - **Target**: Create `StepExecutionResult` and `ExtractionResult` models

#### Implementation Steps

1. Create new Pydantic models in `sentience/models.py`:
   ```python
   class ReadResult(BaseModel):
       status: Literal["success", "error"]
       url: str
       format: Literal["raw", "text", "markdown"]
       content: str
       length: int
       error: Optional[str] = None

   class TraceStats(BaseModel):
       total_steps: int
       total_events: int
       duration_ms: int | None
       final_status: Literal["success", "failure", "partial", "unknown"]
       started_at: str | None
       ended_at: str | None

   class StepExecutionResult(BaseModel):
       success: bool
       action: str
       data: dict[str, Any]  # Can be refined further
       error: Optional[str] = None

   class ExtractionResult(BaseModel):
       found: bool
       data: dict[str, Any]
       summary: str
   ```

2. Update function signatures to return concrete types
3. Update all call sites to use model attributes instead of dict keys
4. Add backward compatibility shims if needed (deprecation warnings)

#### Testing

- ✅ Updated existing tests to use model attributes
- ✅ Added type checking tests using `mypy`
- ✅ Verified backward compatibility (no breaking changes)

#### Completed Models

- ✅ `ReadResult`: For `read()` and `read_async()` return types
- ✅ `TraceStats`: For `get_stats()` methods in `Tracer` and `JsonlTraceSink`
- ✅ `StepExecutionResult`: For `_execute_step()` in `ConversationalAgent`
- ✅ `ExtractionResult`: For `_extract_information()` in `ConversationalAgent`

---

## Phase 2: Code Duplication Reduction

### 2.1 Extract Common Browser Evaluation Patterns

**Priority**: 🟡 Medium  
**Estimated Effort**: 1-2 days  
**Status**: ✅ **Completed 2026-01-02**

#### Issues Identified

- Repeated `browser.page.evaluate()` patterns with similar error handling
- Duplicate logic between sync and async versions of functions

#### Files Affected

- `sentience/read.py` (sync/async duplication)
- `sentience/snapshot.py` (sync/async duplication)
- `sentience/actions.py` (sync/async duplication)
- `sentience/wait.py` (sync/async duplication)

#### Solution

1. ✅ Created `BrowserEvaluator` helper class:
   ```python
   class BrowserEvaluator:
       """Helper for browser page evaluation with consistent error handling"""
       
       @staticmethod
       def invoke(page, method: SentienceMethod | str, *args, **kwargs) -> Any:
           """Invoke window.sentience method synchronously with error handling"""
           
       @staticmethod
       async def invoke_async(page, method: SentienceMethod | str, *args, **kwargs) -> Any:
           """Invoke window.sentience method asynchronously with error handling"""
   ```

2. ✅ Created `SentienceMethod` enum for type-safe method calls:
   - `SNAPSHOT`, `CLICK`, `READ`, `FIND_TEXT_RECT`, `SHOW_OVERLAY`, `CLEAR_OVERLAY`, `START_RECORDING`
   - Integrated into `BrowserEvaluator.invoke()` and `invoke_async()` methods

3. ✅ Created `AgentAction` enum for high-level agent actions:
   - `CLICK`, `TYPE`, `PRESS`, `NAVIGATE`, `SCROLL`, `FINISH`, `WAIT`

4. ✅ Integrated into:
   - `sentience/snapshot.py`: Uses `SentienceMethod.SNAPSHOT`
   - `sentience/text_search.py`: Uses `SentienceMethod.FIND_TEXT_RECT`
   - `sentience/actions.py`: Uses `SentienceMethod.CLICK`

5. ✅ Exported enums from `sentience/__init__.py` for public API

### 2.2 Consolidate Element Filtering Logic

**Priority**: 🟡 Medium  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Issues Identified

- Element filtering logic duplicated across `agent.py`, `base_agent.py`, and `query.py`

#### Solution

1. Create a dedicated `ElementFilter` class:
   ```python
   class ElementFilter:
       """Centralized element filtering logic"""
       
       @staticmethod
       def filter_by_importance(snapshot: Snapshot, max_elements: int = 50) -> list[Element]:
           """Filter elements by importance score"""
           
       @staticmethod
       def filter_by_goal(snapshot: Snapshot, goal: str) -> list[Element]:
           """Filter elements relevant to goal"""
   ```

2. Move filtering logic from `BaseAgent.filter_elements()` to `ElementFilter`
3. Update all call sites to use `ElementFilter`

### 2.3 Extract Common Trace Event Building

**Priority**: 🟡 Medium  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Issues Identified

- Similar trace event building logic in `agent.py` and `agent_async.py`

#### Solution

1. Create `TraceEventBuilder` helper class:
   ```python
   class TraceEventBuilder:
       """Helper for building trace events with consistent structure"""
       
       @staticmethod
       def build_step_end_data(...) -> dict:
           """Build step_end event data"""
           
       @staticmethod
       def build_snapshot_data(...) -> dict:
           """Build snapshot event data"""
   ```

2. Use in both `SentienceAgent` and `SentienceAgentAsync`

---

## Phase 3: Abstraction Improvements

### 3.1 Create Abstract Base Classes for LLM Providers

**Priority**: 🟢 Low  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Current State

- `LLMProvider` is already an abstract base class ✅
- But some providers have duplicate initialization logic

#### Improvements

1. ✅ Created `llm_provider_utils.py` with helper functions:
   - `require_package()`: Consistent ImportError handling for all providers
   - `get_api_key_from_env()`: Standardized API key retrieval from environment variables
   - `handle_provider_error()`: Standardized error handling with provider-specific messages

2. ✅ `LLMResponseBuilder` already exists and is being used ✅

3. ✅ Standardized error handling across all providers:
   - All providers now use `require_package()` for imports (removed duplicate try/except blocks)
   - All providers now use `handle_provider_error()` for API call errors
   - `GeminiProvider` now uses `get_api_key_from_env()` for API key handling

4. ✅ Refactored all 5 LLM providers:
   - `OpenAIProvider`: Uses `require_package()` and `handle_provider_error()`
   - `AnthropicProvider`: Uses `require_package()` and `handle_provider_error()`
   - `GLMProvider`: Uses `require_package()` and `handle_provider_error()`
   - `GeminiProvider`: Uses `require_package()`, `get_api_key_from_env()`, and `handle_provider_error()`
   - `LocalLLMProvider`: Already had proper error handling (no changes needed)

#### Files Updated

- `sentience/llm_provider.py`: Refactored all providers to use `llm_provider_utils` helpers
- `sentience/llm_provider_utils.py`: New helper module for common initialization and error handling
- `tests/test_llm_provider_utils.py`: New comprehensive tests (11 test cases)

### 3.2 Abstract Trace Sink Interface

**Priority**: 🟢 Low  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Current State

- `TraceSink` is already an abstract base class ✅
- But `CloudTraceSink` and `JsonlTraceSink` have some duplicate logic

#### Improvements

1. ✅ `TraceFileManager` already exists and is being used ✅

2. ✅ Extracted common trace stats extraction:
   - Added `TraceFileManager.extract_stats()` method
   - Removed 80+ lines of duplicate code from `JsonlTraceSink.get_stats()`
   - Removed 80+ lines of duplicate code from `CloudTraceSink._extract_stats_from_trace()`
   - Supports custom status inference functions for flexibility

3. ✅ Standardized status inference:
   - Added `TraceFileManager._infer_final_status()` for default inference
   - `CloudTraceSink` uses custom inference that checks run_end events in reverse order
   - Both sinks now use the same core stats extraction logic

4. ✅ Updated both sinks:
   - `JsonlTraceSink.get_stats()`: Now calls `TraceFileManager.extract_stats()` (removed 80+ lines)
   - `CloudTraceSink._extract_stats_from_trace()`: Now calls `TraceFileManager.extract_stats()` with custom inference (removed 80+ lines)

#### Files Updated

- `sentience/trace_file_manager.py`: Extended with `extract_stats()` and `_infer_final_status()` methods
- `sentience/tracing.py`: Refactored `JsonlTraceSink.get_stats()` to use `TraceFileManager.extract_stats()`
- `sentience/cloud_tracing.py`: Refactored `CloudTraceSink._extract_stats_from_trace()` to use `TraceFileManager.extract_stats()`
- `tests/test_trace_file_manager_extract_stats.py`: New comprehensive tests (9 test cases)

---

## Phase 4: Modular Structure Improvements

### 4.1 Reorganize Utility Functions

**Priority**: 🟡 Medium  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Current Issues

- Utility functions scattered across multiple files
- Some utilities are file-specific but could be shared

#### Solution

1. ✅ Created `sentience/utils/` package:
   ```
   sentience/utils/
   ├── __init__.py          # Re-exports all functions for backward compatibility
   ├── browser.py           # Browser-related utilities (save_storage_state)
   ├── element.py           # Element manipulation utilities (digests, normalization)
   └── formatting.py        # Text formatting utilities (format_snapshot_for_llm)
   ```

2. ✅ Moved functions from:
   - `utils.py` → `sentience/utils/element.py` and `sentience/utils/browser.py`
   - `formatting.py` → `sentience/utils/formatting.py`
   - All element digest utilities consolidated in `utils/element.py`

3. ✅ **Maintained backward compatibility**:
   - `sentience/utils/__init__.py` re-exports all functions from submodules
   - `sentience/__init__.py` imports from new locations via `utils/__init__.py`
   - Users can continue using: `from sentience import canonical_snapshot_strict, ...`
   - Users can continue using: `from sentience.utils import compute_snapshot_digests, ...`
   - **No breaking changes to public API** - all tests pass

#### Files Updated

- `sentience/utils/__init__.py`: New module with re-exports for backward compatibility
- `sentience/utils/browser.py`: Browser utilities (save_storage_state)
- `sentience/utils/element.py`: Element digest utilities (canonical_snapshot_*, compute_snapshot_digests, etc.)
- `sentience/utils/formatting.py`: Formatting utilities (format_snapshot_for_llm)
- `sentience/__init__.py`: Updated imports to use new utils package structure
- `sentience/element_filter.py`: Fixed type hint to use `Optional[str]` (Phase 1.0 compliance)

### 4.2 Separate Concerns in Agent Classes

**Priority**: 🟡 Medium  
**Estimated Effort**: 2 days  
**Status**: ✅ **Completed 2026-01-02**

#### Current Issues

- `SentienceAgent` and `SentienceAgentAsync` are large (1500+ lines)
- Mixing concerns: LLM interaction, action execution, trace building

#### Solution

1. ✅ Created `LLMInteractionHandler` class (`sentience/llm_interaction_handler.py`):
   - `build_context()`: Formats snapshot elements for LLM context
   - `query_llm()`: Queries LLM with standardized prompt template
   - `extract_action()`: Parses action command from LLM response
   - Encapsulates all LLM interaction logic, making it easier to test and modify

2. ✅ Created `ActionExecutor` class (`sentience/action_executor.py`):
   - `execute()`: Parses and executes action strings (synchronous)
   - `execute_async()`: Parses and executes action strings (asynchronous)
   - Handles CLICK, TYPE, PRESS, and FINISH actions
   - Detects browser type (sync/async) and raises appropriate errors

3. ✅ Trace building already extracted to `TraceEventBuilder` (completed in Phase 2.3)

4. ✅ Refactored `SentienceAgent` and `SentienceAgentAsync`:
   - Removed `_build_context()`, `_query_llm()`, `_extract_action_from_response()`, and `_execute_action()` methods
   - Initialize handlers in `__init__`: `self.llm_handler` and `self.action_executor`
   - Updated `act()` methods to use handlers instead of internal methods
   - Reduced code duplication between sync and async versions

#### Files Created

- `sentience/llm_interaction_handler.py`: LLM interaction handler (120 lines)
- `sentience/action_executor.py`: Action execution handler (180 lines)

#### Files Updated

- `sentience/agent.py`: Removed 200+ lines of duplicated handler logic, now uses handlers
- `tests/test_agent.py`: Updated tests to use handlers instead of private methods

#### Benefits

- **Separation of Concerns**: LLM interaction, action execution, and trace building are now separate
- **Testability**: Handlers can be tested independently
- **Maintainability**: Changes to LLM prompts or action parsing are centralized
- **Code Reduction**: Removed ~200 lines of duplicated code from agent classes
- **No Breaking Changes**: Public API remains unchanged, all tests pass

#### Backward Compatibility

- **No impact on user imports**: `LLMInteractionHandler` and `ActionExecutor` are **internal implementation details**
- Users continue to use: `from sentience import SentienceAgent, SentienceAgentAsync`
- The public API (`SentienceAgent`, `SentienceAgentAsync`) remains unchanged
- Only internal code organization changes

---

## Phase 5: Testability Improvements

### 5.1 Improve Mockability

**Priority**: 🔴 High  
**Estimated Effort**: 2-3 days  
**Status**: ✅ **Completed 2026-01-02**

#### Issues Identified

- Hard dependencies on `SentienceBrowser` and `Playwright` page objects
- Difficult to test without real browser instances
- Current mocks are too basic (can't test error conditions, timeouts, edge cases)
- Only 1-2 error handling tests exist (retry logic, invalid action)
- Missing tests for: network failures, timeouts, browser crashes, state errors

#### Solution

1. Created `BrowserProtocol` and `PageProtocol` with `@runtime_checkable` decorator:
   - `BrowserProtocol`: Defines minimal interface for browser operations
   - `PageProtocol`: Defines minimal interface for page operations
   - `AsyncBrowserProtocol` and `AsyncPageProtocol`: Async versions

2. Updated classes to accept protocol types:
   - `SentienceAgent`: Accepts `Union[SentienceBrowser, BrowserProtocol]`
   - `SentienceAgentAsync`: Accepts `Union[AsyncSentienceBrowser, AsyncBrowserProtocol]`
   - `ActionExecutor`: Accepts protocol types, with improved async detection
   - `ConversationalAgent`: Accepts `Union[SentienceBrowser, BrowserProtocol]`

3. Created mock implementations:
   - `MockBrowser`: Implements `BrowserProtocol` for unit testing
   - `MockPage`: Implements `PageProtocol` with proper snapshot response format
   - `MockLLMProvider`: Implements `LLMProvider` with configurable responses

4. Fixed async detection in `ActionExecutor`:
   - Uses `inspect.iscoroutinefunction()` to check if methods are actually async
   - Prevents `MockBrowser` from being incorrectly detected as async

5. Added graceful tracer error handling:
   - Created `_safe_tracer_call()` helper function
   - Wrapped all tracer calls to prevent tracer errors from breaking agent execution

#### Benefits

- **Test Error Conditions**: Can simulate network failures, timeouts, browser crashes
- **Faster Tests**: Unit tests with mocks (<0.1s) vs integration tests (2-5s)
- **Better Coverage**: Enables 20-30 new focused unit tests
- **Test Isolation**: Focus on agent logic, not browser quirks

#### Implementation Details

- **Protocols**: Created in `sdk-python/sentience/protocols.py`
- **Mock Implementations**: Created in `sdk-python/tests/unit/test_agent_errors.py`
- **Test Organization**: Created `tests/unit/` and `tests/integration/` directories
- **Backward Compatibility**: `SentienceBrowser` naturally implements `BrowserProtocol`, no changes needed

#### Test Results

- **13 new unit tests** added for error handling and edge cases
- **13/13 tests passing** ✅
- **All existing tests pass** (15 passed, 2 skipped)
- **Test Categories**:
  - **Error handling** (8 tests): snapshot timeout, network failure, action timeout, browser not started, empty snapshot, malformed LLM response, URL change during action, retry on transient error
  - **Edge cases** (5 tests): zero elements in snapshot, unicode in actions, special characters in goal, state preservation on retry, tracer errors graceful handling

### 5.2 Add Dependency Injection

**Priority**: 🟡 Medium  
**Estimated Effort**: 1-2 days  
**Status**: ✅ **Completed 2026-01-02**

#### Solution

1. Refactored constructors to accept protocol types:
   - `SentienceAgent.__init__`: Accepts `Union[SentienceBrowser, BrowserProtocol]`
   - `SentienceAgentAsync.__init__`: Accepts `Union[AsyncSentienceBrowser, AsyncBrowserProtocol]`
   - `ConversationalAgent.__init__`: Accepts `Union[SentienceBrowser, BrowserProtocol]`
   - `ActionExecutor.__init__`: Accepts protocol types with improved async detection

2. Maintained backward compatibility:
   - All existing code continues to work (no breaking changes)
   - `SentienceBrowser` naturally implements `BrowserProtocol`
   - Type hints use `Union` to support both concrete and protocol types

3. Updated tests to use dependency injection:
   - Created `MockBrowser` and `MockPage` for unit testing
   - All new unit tests use protocol-compatible mocks
   - Existing integration tests continue to use real browsers

#### Benefits

- **Better Testability**: Can inject mocks for isolated unit testing
- **Type Safety**: Protocol types provide compile-time type checking
- **Flexibility**: Supports both concrete types and protocol-compatible objects
- **No Breaking Changes**: Existing code continues to work without modification

#### Implementation Details

- **Protocol Types**: All agent constructors now accept `Union[ConcreteType, ProtocolType]`
- **Async Detection**: Fixed in `ActionExecutor` using `inspect.iscoroutinefunction()` to check actual method signatures
- **Tracer Error Handling**: All tracer calls wrapped in `_safe_tracer_call()` helper to prevent tracer errors from breaking agent execution

#### See Also

- `docs/PHASE_5_ANALYSIS.md` - Detailed analysis of benefits, risks, and test coverage impact

### 5.2 Add Dependency Injection

**Priority**: 🟡 Medium  
**Estimated Effort**: 1-2 days

#### Solution

1. Refactor constructors to accept dependencies:
   ```python
   class SentienceAgent:
       def __init__(
           self,
           browser: BrowserProtocol,
           llm: LLMProvider,
           tracer: Tracer | None = None,
           config: AgentConfig | None = None,
       ):
   ```

2. Create factory functions for common configurations
3. Update tests to use dependency injection

### 5.3 Improve Test Coverage

**Priority**: 🟡 Medium  
**Estimated Effort**: Ongoing  
**Status**: ✅ **Completed 2026-01-02**

#### Actions

1. ✅ Add unit tests for utility functions
   - Added 7 tests for `save_storage_state` in `utils/browser.py` (`tests/test_utils_browser.py`)
   - Coverage for `utils/browser.py` increased from 40% to 100%
   - Tests cover: file creation, parent directory creation, string/Path paths, JSON formatting, empty state, success messages

2. ✅ Add integration tests for agent workflows
   - Created `tests/integration/test_agent_workflows.py` with 10 integration tests
   - Tests cover: multi-step workflows, error recovery, state management, token tracking
   - Test categories:
     - **Multi-step workflows** (5 tests): click+type sequences, retry scenarios, URL changes, finish actions, token tracking
     - **Error recovery** (3 tests): snapshot failure recovery, action failure recovery, max retries exceeded
     - **State management** (2 tests): history preservation, step count increments

3. ⏳ Add property-based tests for edge cases
   - **Pending**: Consider adding `hypothesis` for property-based testing
   - Focus areas: text normalization edge cases, bbox normalization edge cases, element fingerprint extraction

4. ⏳ Set coverage target: 80% for core modules
   - **Current overall coverage**: 64%
   - **Target modules needing improvement**:
     - `overlay.py`: 48% (needs tests)
     - `read.py`: 49% (needs tests)
     - `text_search.py`: 39% (needs tests)
     - `snapshot.py`: 32% (needs tests)
     - `recorder.py`: 65% (needs more tests)
     - `query.py`: 66% (needs more tests)

#### Test Organization

- **Unit tests**: `tests/unit/` - Fast, isolated tests with mocks
- **Integration tests**: `tests/integration/` - Multi-step workflows and error recovery
- **Existing tests**: `tests/` - Legacy location (maintained for backward compatibility)

#### Files Created

- `tests/test_utils_browser.py`: 7 unit tests for `save_storage_state`
- `tests/integration/test_agent_workflows.py`: 10 integration tests for agent workflows

---

## Phase 6: Code Linting and Style

### 6.1 Set Up Pre-commit Hooks

**Priority**: 🔴 High  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Implementation

1. Install pre-commit:
   ```bash
   pip install pre-commit
   ```

2. Create `.pre-commit-config.yaml`:
   ```yaml
   repos:
     - repo: https://github.com/pre-commit/pre-commit-hooks
       rev: v4.5.0
       hooks:
         - id: trailing-whitespace
         - id: end-of-file-fixer
         - id: check-yaml
         - id: check-added-large-files
         - id: check-json
         - id: check-toml
         - id: check-merge-conflict
         - id: debug-statements
         
     - repo: https://github.com/psf/black
       rev: 23.12.1
       hooks:
         - id: black
           language_version: python3.11
           
     - repo: https://github.com/pycqa/isort
       rev: 5.13.2
       hooks:
         - id: isort
           args: ["--profile", "black"]
           
     - repo: https://github.com/pycqa/flake8
       rev: 7.0.0
       hooks:
         - id: flake8
           args: ["--max-line-length=100", "--extend-ignore=E203,W503,E501"]
           
     - repo: https://github.com/pre-commit/mirrors-mypy
       rev: v1.8.0
       hooks:
         - id: mypy
           args: ["--ignore-missing-imports"]
           additional_dependencies: [types-all]
   ```

3. Install hooks:
   ```bash
   pre-commit install
   ```

### 6.2 Update GitHub Actions

**Priority**: 🔴 High  
**Estimated Effort**: 1 day  
**Status**: ✅ **Completed 2026-01-02**

#### Update `.github/workflows/test.yml`

Add linting step:
```yaml
- name: Lint with pre-commit
  run: |
    pip install pre-commit
    pre-commit run --all-files

- name: Type check with mypy
  run: |
    pip install mypy types-all
    mypy sentience --ignore-missing-imports

- name: Check code style
  run: |
    pip install black isort flake8
    black --check sentience tests
    isort --check-only --profile black sentience tests
    flake8 sentience tests --max-line-length=100 --extend-ignore=E203,W503,E501
```

### 6.3 Code Style Guidelines

**Priority**: 🟡 Medium  
**Estimated Effort**: Ongoing

#### Document Style Guide

1. Create `docs/STYLE_GUIDE.md`:
   - Naming conventions
   - Function/method organization
   - Docstring format (Google style)
   - Type hint requirements

2. Enforce via pre-commit and CI

---

## Phase 7: Clean Code Principles

### 7.1 Improve Function Naming

**Priority**: 🟡 Medium  
**Estimated Effort**: 1 day

#### Issues

- Some functions have unclear names
- Inconsistent naming patterns

#### Actions

1. Audit function names for clarity
2. Rename functions to follow Python conventions:
   - Functions: `snake_case`
   - Classes: `PascalCase`
   - Constants: `UPPER_SNAKE_CASE`

### 7.2 Improve Documentation

**Priority**: 🟡 Medium  
**Estimated Effort**: 2 days

#### Actions

1. Add docstrings to all public functions/classes
2. Use Google-style docstrings
3. Add type hints to all function signatures
4. Document complex algorithms

### 7.3 Reduce Function Complexity

**Priority**: 🟡 Medium  
**Estimated Effort**: 2-3 days

#### Issues

- Some functions are too long (>100 lines)
- High cyclomatic complexity

#### Actions

1. Identify functions with complexity > 15 (flake8 max-complexity)
2. Refactor into smaller functions
3. Extract complex conditionals into helper functions

---

## Implementation Timeline

### Week 1: Foundation
- ✅ Phase 1.0: Standardize Optional Type Hints (High Priority) - **Completed 2026-01-02**
- ✅ Phase 1.1: Replace `dict` return types (High Priority) - **Completed 2026-01-02**
- ✅ Phase 6.1-6.2: Set up linting (High Priority) - **Completed 2026-01-02**

### Week 2: Code Quality
- ✅ Phase 2.1: Extract Common Browser Evaluation Patterns (Medium Priority) - **Completed 2026-01-02**
  - Created `BrowserEvaluator` helper class with `invoke()` and `invoke_async()` methods
  - Created `SentienceMethod` enum for type-safe window.sentience API method calls
  - Created `AgentAction` enum for high-level agent action types
  - Integrated into `snapshot.py`, `text_search.py`, and `actions.py`
- ✅ Phase 2.2: Consolidate Element Filtering Logic (Medium Priority) - **Completed 2026-01-02**
  - Created `ElementFilter` class with `filter_by_importance()` and `filter_by_goal()` methods
  - Refactored both `SentienceAgent` and `SentienceAgentAsync` to use centralized filtering
  - Removed 160+ lines of duplicate code
- ✅ Phase 2.3: Extract Common Trace Event Building (Medium Priority) - **Completed 2026-01-02**
  - Created `TraceEventBuilder` class with `build_snapshot_event()` and `build_step_end_event()` methods
  - Refactored both sync and async agents to use centralized event building
  - Removed duplicate trace event building logic (6 occurrences)
- ⏳ Phase 7.1-7.2: Improve naming and documentation (Medium Priority) - **Pending**

### Week 3: Architecture
- ✅ Phase 3.1: Create Abstract Base Classes for LLM Providers (Low Priority) - **Completed 2026-01-02**
  - Created `llm_provider_utils.py` with `require_package()`, `get_api_key_from_env()`, and `handle_provider_error()`
  - Refactored all 5 providers (OpenAI, Anthropic, GLM, Gemini, LocalLLM) to use standardized initialization and error handling
  - Removed duplicate ImportError handling and error handling code
  - Added comprehensive tests in `tests/test_llm_provider_utils.py` (11 test cases)
- ✅ Phase 3.2: Abstract Trace Sink Interface (Low Priority) - **Completed 2026-01-02**
  - Extended `TraceFileManager` with `extract_stats()` method
  - Removed 160+ lines of duplicate stats extraction code from both sinks
  - Standardized status inference logic with support for custom inference functions
  - Added comprehensive tests in `tests/test_trace_file_manager_extract_stats.py` (9 test cases)
- ✅ Phase 4.1: Reorganize Utility Functions (Medium Priority) - **Completed 2026-01-02**
  - Created `sentience/utils/` package with submodules (browser.py, element.py, formatting.py)
  - Maintained full backward compatibility via `__init__.py` re-exports
  - All 322 tests passing, no breaking changes
- ✅ Phase 4.2: Separate Concerns in Agent Classes (Medium Priority) - **Completed 2026-01-02**
  - Created `LLMInteractionHandler` class for LLM interaction logic
  - Created `ActionExecutor` class for action execution logic
  - Refactored both `SentienceAgent` and `SentienceAgentAsync` to use handlers
  - Removed 200+ lines of duplicated code, all 15 agent tests passing

### Week 4: Testing
- ✅ Phase 5.1-5.3: Improve testability (High Priority)
- ✅ Phase 7.3: Reduce complexity (Medium Priority)

---

## Success Metrics

1. **Type Safety**: 100% of public functions return concrete types (no `dict`)
2. **Code Duplication**: < 5% duplicate code (measured by tools)
3. **Test Coverage**: > 80% for core modules
4. **Linting**: 0 linting errors in CI
5. **Complexity**: All functions < 15 cyclomatic complexity
6. **Documentation**: 100% of public APIs documented

---

## Risk Mitigation

1. **Backward Compatibility**: Add deprecation warnings for breaking changes
2. **Incremental Changes**: Implement changes in phases to avoid large refactors
3. **Testing**: Maintain test coverage during refactoring
4. **Code Review**: All changes require peer review

---

## Related Documentation

- `docs/STYLE_GUIDE.md` - Code style guidelines (to be created)
- `pyproject.toml` - Linting configuration
- `.pre-commit-config.yaml` - Pre-commit hooks (to be created)

---

*Last updated: 2026-01-02*

---

## Progress Summary

### Completed Phases ✅

1. **Phase 1.0**: Standardized Optional Type Hints (124 instances across 17 files) - **Completed 2026-01-02**
2. **Phase 1.1**: Replaced `dict` return types with Pydantic models (`ReadResult`, `TraceStats`, `StepExecutionResult`, `ExtractionResult`) - **Completed 2026-01-02**
3. **Phase 2.1**: Created `BrowserEvaluator` helper class and `SentienceMethod`/`AgentAction` enums - **Completed 2026-01-02**
4. **Phase 2.2**: Created `ElementFilter` class and consolidated element filtering logic - **Completed 2026-01-02**
5. **Phase 2.3**: Created `TraceEventBuilder` class and extracted common trace event building - **Completed 2026-01-02**
6. **Phase 3.1**: Created `llm_provider_utils.py` and standardized LLM provider initialization/error handling - **Completed 2026-01-02**
7. **Phase 3.2**: Extended `TraceFileManager` with `extract_stats()` and removed duplicate stats extraction code - **Completed 2026-01-02**
8. **Phase 4.1**: Reorganized utility functions into `sentience/utils/` package with full backward compatibility - **Completed 2026-01-02**
9. **Phase 4.2**: Separated concerns in agent classes by creating `LLMInteractionHandler` and `ActionExecutor` - **Completed 2026-01-02**
10. **Phase 6.1-6.2**: Set up pre-commit hooks and GitHub Actions linting - **Completed 2026-01-02**

### In Progress 🚧

- None currently

### Pending ⏳

- Phase 5: Testability Improvements
- Phase 7: Clean Code Principles

