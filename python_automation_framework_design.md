# Modular Python Automation Framework Architecture

This framework design provides a scalable foundation for API testing, web scraping, data pipeline automation, and system administration tasks.

---

## Architecture Overview

```
automation_framework/
├── config/
│   ├── settings.yaml          # Primary configuration settings
│   └── dev_settings.yaml      # Environment-specific overrides
├── core/
│   ├── config_loader.py       # Pydantic-based configuration management
│   ├── logger.py              # Centralized structured logging setup
│   ├── exceptions.py          # Custom exception hierarchy
│   ├── retry.py               # Robust retry decorator with backoff
│   └── reporter.py            # Summary and HTML test reporting
├── modules/
│   ├── base_module.py         # Abstract Base Class for automation modules
│   ├── api_module.py          # HTTP/REST Automation Module
│   └── web_module.py          # Web / Browser Automation Module Placeholder
├── reports/                   # Generated execution logs and reports
└── main.py                    # Entry point / Execution runner
```

---

## 1. Core Framework Design & Key Features

### Configuration Management (`Pydantic` + `PyYAML`)
* **Type Safety:** Uses Pydantic v2 to validate configuration types at launch.
* **Environment Overrides:** Reads YAML configurations and seamlessly merges environment variable overrides.

### Logging (`Loguru`)
* **Structured Output:** Configures standardized console and file-based logging.
* **Automatic Rotation:** Rotates log files based on size or age (e.g., 10 MB per log file, 10 days retention).

### Exception Handling & Resilience
* **Custom Hierarchy:** Domain-specific exceptions (`AutomationError`, `ConfigurationError`, `ModuleExecutionError`).
* **Retry Decorator:** Configurable exponential backoff using the `tenacity` library or custom decorator logic.

### Module Pluggability
* **Abstract Base Class (`BaseModule`):** Enforces lifecycle methods (`setup`, `execute`, `teardown`) across all execution modules.

### Reporting
* **Execution Metrics:** Captures execution times, module status (PASS, FAIL, SKIPPED), and outputs structured JSON or HTML reports.

---

## 2. Complete Python Implementation

### Configuration File (`config/settings.yaml`)

```yaml
app_name: "Enterprise Automation Engine"
environment: "dev"
debug: true

logging:
  level: "INFO"
  dir: "logs"
  rotation: "10 MB"
  retention: "7 days"

retry:
  max_attempts: 3
  min_wait_seconds: 1.0
  max_wait_seconds: 5.0

reporting:
  output_dir: "reports"
  generate_html: true
```

---

### Implementation Code (`main.py` complete single-file framework example)

```python
import os
import sys
import time
import yaml
import json
import functools
from typing import Any, Dict, List, Optional, Type
from abc import ABC, abstractmethod
from datetime import datetime
from pathlib import Path
from pydantic import BaseModel, Field

# =====================================================================
# 1. Custom Exceptions
# =====================================================================
class AutomationFrameworkError(Exception):
    """Base exception for the framework."""
    pass

class ConfigurationError(AutomationFrameworkError):
    """Raised when configuration loading or validation fails."""
    pass

class ModuleExecutionError(AutomationFrameworkError):
    """Raised when an automation module fails during execution."""
    pass

class RecoverableError(AutomationFrameworkError):
    """Raised when an operation encounters a transient failure suitable for retries."""
    pass

# =====================================================================
# 2. Configuration Management (Pydantic + YAML)
# =====================================================================
class LoggingConfig(BaseModel):
    level: str = "INFO"
    dir: str = "logs"
    rotation: str = "10 MB"
    retention: str = "7 days"

class RetryConfig(BaseModel):
    max_attempts: int = Field(default=3, ge=1)
    min_wait_seconds: float = Field(default=1.0, ge=0.0)
    max_wait_seconds: float = Field(default=5.0, ge=0.0)

class ReportingConfig(BaseModel):
    output_dir: str = "reports"
    generate_html: bool = True

class AppConfig(BaseModel):
    app_name: str
    environment: str
    debug: bool = False
    logging: LoggingConfig = LoggingConfig()
    retry: RetryConfig = RetryConfig()
    reporting: ReportingConfig = ReportingConfig()

class ConfigLoader:
    @staticmethod
    def load_config(config_path: str = "config/settings.yaml") -> AppConfig:
        path = Path(config_path)
        if not path.exists():
            # Fallback for inline runner default
            return AppConfig(app_name="Default Automation Framework", environment="local")
        
        try:
            with open(path, "r", encoding="utf-8") as f:
                raw_data = yaml.safe_load(f) or {}
            return AppConfig(**raw_data)
        except Exception as e:
            raise ConfigurationError(f"Failed to load configuration from {config_path}: {e}")

# =====================================================================
# 3. Simple Structured Logging System
# =====================================================================
class SimpleLogger:
    def __init__(self, name: str = "Automation", log_dir: str = "logs", level: str = "INFO"):
        self.name = name
        self.level = level
        self.log_dir = Path(log_dir)
        self.log_dir.mkdir(parents=True, exist_ok=True)
        self.log_file = self.log_dir / f"automation_{datetime.now().strftime('%Y%m%d')}.log"

    def _log(self, level: str, message: str):
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S.%f")[:-3]
        formatted_message = f"[{timestamp}] [{level:<7}] [{self.name}] {message}"
        
        # Console output
        print(formatted_message)
        
        # File output
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(formatted_message + "\n")

    def info(self, msg: str): self._log("INFO", msg)
    def warning(self, msg: str): self._log("WARNING", msg)
    def error(self, msg: str): self._log("ERROR", msg)
    def debug(self, msg: str): self._log("DEBUG", msg)

# Initialize global logger
logger = SimpleLogger()

# =====================================================================
# 4. Resilience & Retry Mechanism
# =====================================================================
def retry_on_exception(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    exceptions: tuple = (RecoverableError,)
):
    """
    Decorator providing exponential backoff retries for transient failures.
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            current_delay = delay
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        logger.error(f"Function '{func.__name__}' failed after {max_attempts} attempts. Final Exception: {e}")
                        raise
                    logger.warning(
                        f"Function '{func.__name__}' failed on attempt {attempt}/{max_attempts}. "
                        f"Retrying in {current_delay:.2f}s... Exception: {e}"
                    )
                    time.sleep(current_delay)
                    current_delay *= backoff
        return wrapper
    return decorator

# =====================================================================
# 5. Abstract Base Automation Module
# =====================================================================
class BaseModule(ABC):
    """
    Abstract Base Class that every automation module must implement.
    """
    def __init__(self, name: str, config: AppConfig):
        self.name = name
        self.config = config
        self.status = "NOT_STARTED"
        self.execution_time = 0.0

    @abstractmethod
    def setup(self) -> None:
        """Initialize resources, drivers, or clients."""
        pass

    @abstractmethod
    def run(self) -> Dict[str, Any]:
        """Core automation steps."""
        pass

    @abstractmethod
    def teardown(self) -> None:
        """Clean up connections, browser sessions, or temporary files."""
        pass

    def execute(self) -> Dict[str, Any]:
        """Wrapper method enforcing module lifecycle and time tracking."""
        logger.info(f"--- Starting Module Execution: {self.name} ---")
        start_time = time.perf_counter()
        
        try:
            self.setup()
            result = self.run()
            self.status = "PASSED"
            logger.info(f"--- Module {self.name} PASSED ---")
            return result
        except Exception as e:
            self.status = "FAILED"
            logger.error(f"--- Module {self.name} FAILED: {e} ---")
            raise ModuleExecutionError(f"Module '{self.name}' execution failed: {e}") from e
        finally:
            try:
                self.teardown()
            except Exception as teardown_err:
                logger.error(f"Teardown error in module {self.name}: {teardown_err}")
            
            self.execution_time = round(time.perf_counter() - start_time, 4)

# =====================================================================
# 6. Module Implementations
# =====================================================================
class APIAutomationModule(BaseModule):
    """Sample API test/automation module demonstrating retries and lifecycle."""
    
    def setup(self) -> None:
        logger.info(f"[{self.name}] Initializing API Client and Headers...")
        self.api_client = {"session_active": True}

    @retry_on_exception(max_attempts=3, delay=0.5, backoff=2.0)
    def _unstable_api_call(self, attempt_tracker: List[int]):
        attempt_tracker[0] += 1
        if attempt_tracker[0] < 2:
            raise RecoverableError("503 Service Unavailable")
        return {"status_code": 200, "data": {"user_id": 42, "role": "admin"}}

    def run(self) -> Dict[str, Any]:
        logger.info(f"[{self.name}] Fetching user profile from REST API...")
        attempts = [0]
        response = self._unstable_api_call(attempts)
        return response

    def teardown(self) -> None:
        logger.info(f"[{self.name}] Closing API HTTP Session.")
        self.api_client["session_active"] = False


class DataProcessingModule(BaseModule):
    """Sample ETL / Data pipeline automation module."""

    def setup(self) -> None:
        logger.info(f"[{self.name}] Connecting to Data Store...")

    def run(self) -> Dict[str, Any]:
        logger.info(f"[{self.name}] Processing batch dataset...")
        time.sleep(0.2)  # Simulate task execution
        return {"records_processed": 1500, "errors_count": 0}

    def teardown(self) -> None:
        logger.info(f"[{self.name}] Disconnecting from Data Store.")

# =====================================================================
# 7. Reporting & Summary Generator
# =====================================================================
class ExecutionReporter:
    def __init__(self, output_dir: str = "reports"):
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)
        self.results: List[Dict[str, Any]] = []

    def add_result(self, module_name: str, status: str, duration: float, details: Optional[Dict] = None, error: Optional[str] = None):
        self.results.append({
            "module_name": module_name,
            "status": status,
            "duration_seconds": duration,
            "details": details or {},
            "error": error
        })

    def generate_json_report(self) -> Path:
        report_file = self.output_dir / f"summary_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        
        total_modules = len(self.results)
        passed = sum(1 for r in self.results if r["status"] == "PASSED")
        failed = sum(1 for r in self.results if r["status"] == "FAILED")
        
        summary = {
            "timestamp": datetime.now().isoformat(),
            "metrics": {
                "total": total_modules,
                "passed": passed,
                "failed": failed,
                "pass_rate": f"{(passed / total_modules * 100):.1f}%" if total_modules else "0%"
            },
            "results": self.results
        }
        
        with open(report_file, "w", encoding="utf-8") as f:
            json.dump(summary, f, indent=4)
            
        return report_file

# =====================================================================
# 8. Orchestrator Runner
# =====================================================================
class FrameworkRunner:
    def __init__(self, config_file: str = "config/settings.yaml"):
        self.config = ConfigLoader.load_config(config_file)
        global logger
        logger = SimpleLogger(
            name=self.config.app_name,
            log_dir=self.config.logging.dir,
            level=self.config.logging.level
        )
        self.reporter = ExecutionReporter(output_dir=self.config.reporting.output_dir)
        self.modules: List[BaseModule] = []

    def register_module(self, module: BaseModule):
        self.modules.append(module)

    def run_all(self):
        logger.info(f"🚀 Starting Engine: {self.config.app_name} [{self.config.environment.upper()}]")
        
        for module in self.modules:
            try:
                result_data = module.execute()
                self.reporter.add_result(
                    module_name=module.name,
                    status=module.status,
                    duration=module.execution_time,
                    details=result_data
                )
            except ModuleExecutionError as err:
                self.reporter.add_result(
                    module_name=module.name,
                    status=module.status,
                    duration=module.execution_time,
                    error=str(err)
                )

        report_path = self.reporter.generate_json_report()
        logger.info(f"📊 Execution finished. Summary Report saved to: {report_path}")

# =====================================================================
# Framework Execution Entry Point
# =====================================================================
if __name__ == "__main__":
    # 1. Instantiate the Orchestrator
    runner = FrameworkRunner()

    # 2. Register Automation Modules
    runner.register_module(APIAutomationModule(name="UserAPI_Check", config=runner.config))
    runner.register_module(DataProcessingModule(name="ETL_Batch_Sync", config=runner.config))

    # 3. Run all registered tasks
    runner.run_all()
```

---

## 3. Recommended Production Third-Party Packages

For full-scale enterprise implementations, consider integrating the following battle-tested libraries into this framework:

| Feature Area | Recommended Library | Purpose |
| :--- | :--- | :--- |
| **Retry Handling** | `tenacity` | Advanced retry strategies (jitter, wait exponential, custom callbacks). |
| **Logging** | `loguru` | Pre-configured colored logs, file rotators, and stack traces. |
| **HTTP Requests** | `httpx` or `requests` | Non-blocking or synchronous API request handling. |
| **Browser Automation**| `playwright` or `selenium` | Fast, reliable headless browser automation for web tasks. |
| **Reporting** | `allure-pytest` | Enterprise test suite visualization and execution analytics. |