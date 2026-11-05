# Processing Multi-Gigabyte Log Files efficiently in Python

Processing 10 GB to 100+ GB log files without running out of memory (OOM) or exhausting local disk space requires streaming paradigms, generator pipelines, and zero-copy memory management.

---

## 1. Core Architectural Strategies

### Memory Efficiency: Streaming Line-by-Line with Generators
Never use `file.read()` or `file.readlines()` on large files as they attempt to read the entire file into RAM. Python’s file object is an **iterator that yields lines lazily**, keeping memory consumption below a few megabytes regardless of file size.

* **RAM Footprint:** $O(1)$ constant memory usage relative to file size.

### Disk Efficiency: Direct Compressed Stream Reading
Log files in production are typically stored compressed (e.g., `.gz` or `.bz2`). Instead of decompressing a 100 GB file onto disk (which requires an extra 100 GB of storage space), stream and decompress the file directly in memory using Python's built-in `gzip` or `bz2` modules.

### High-Performance Pattern Matching
* **Pre-compiled Regex:** Compile regular expressions using `re.compile()` once outside loop iterations.
* **Fast Path Filtering:** Perform string membership checks (e.g., `if "ERROR" in line:`) *before* executing expensive regex evaluations.

### Multi-Core Parallel Processing
For 100+ GB uncompressed files, a single CPU core may be bottlenecked by string parsing speeds. Split the file reading across process pools using **chunking / offset boundaries**:
1. Determine file size in bytes.
2. Divide the file into $N$ roughly equal byte ranges (where $N$ is CPU count).
3. Seek to each range start, skip to the next newline (`\n`), process the chunk, and aggregate results.

---

## 2. Complete Python Implementation

Below is a production-grade script that streams `.log` or `.gz` files, extracts error metrics, tracks top failing endpoints, and outputs summary statistics with minimal memory usage.

```python
import gzip
import re
import argparse
import time
from collections import Counter
from typing import Generator, Dict, Any, TextIO
from pathlib import Path


# Pre-compile regular expressions for efficiency
TIMESTAMP_PATTERN = re.compile(r'^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}')
ERROR_PATTERN = re.compile(r'ERROR\s+\[(?P<module>[\w\.]+)\]\s+(?P<msg>.*)')
ENDPOINT_PATTERN = re.compile(r'(GET|POST|PUT|DELETE|PATCH)\s+(?P<path>/[^\s\?]*)')


def stream_file_lines(file_path: Path) -> Generator[str, None, None]:
    """
    Generator function that streams lines lazily from plain text or gzipped log files.
    Ensures $O(1)$ memory consumption regardless of file size.
    """
    open_fn = gzip.open if file_path.suffix == '.gz' else open
    mode = 'rt' if file_path.suffix == '.gz' else 'r'

    # Using a 1MB read buffer size for optimal disk I/O performance
    buffer_size = 1024 * 1024 
    
    with open_fn(file_path, mode=mode, encoding='utf-8', errors='replace', buffering=buffer_size) as f:
        for line in f:
            yield line


def process_large_log(file_path: Path) -> Dict[str, Any]:
    """
    Processes a massive log file line-by-line without loading it into RAM.
    Accumulates counts and frequency distributions.
    """
    total_lines = 0
    error_count = 0
    warning_count = 0
    info_count = 0
    
    # Track top failing modules and endpoints
    module_errors = Counter()
    endpoint_failures = Counter()

    start_time = time.perf_counter()

    for line in stream_file_lines(file_path):
        total_lines += 1

        # Fast-path check: avoid expensive regex if keyword doesn't exist
        if "ERROR" in line:
            error_count += 1
            
            # Match module name
            match = ERROR_PATTERN.search(line)
            if match:
                module = match.group("module")
                module_errors[module] += 1
            
            # Match associated endpoint if available
            ep_match = ENDPOINT_PATTERN.search(line)
            if ep_match:
                endpoint = f"{ep_match.group(1)} {ep_match.group('path')}"
                endpoint_failures[endpoint] += 1

        elif "WARN" in line or "WARNING" in line:
            warning_count += 1
        elif "INFO" in line:
            info_count += 1

    elapsed_time = time.perf_counter() - start_time

    return {
        "file_name": file_path.name,
        "file_size_gb": round(file_path.stat().st_size / (1024 ** 3), 3),
        "total_lines": total_lines,
        "execution_time_seconds": round(elapsed_time, 2),
        "lines_per_second": int(total_lines / elapsed_time) if elapsed_time > 0 else 0,
        "log_levels": {
            "ERROR": error_count,
            "WARNING": warning_count,
            "INFO": info_count
        },
        "top_error_modules": module_errors.most_common(5),
        "top_failing_endpoints": endpoint_failures.most_common(5)
    }


def main():
    parser = argparse.ArgumentParser(description="Analyze huge log files efficiently.")
    parser.add_argument("log_file", type=str, help="Path to .log or .log.gz file")
    args = parser.parse_args()

    path = Path(args.log_file)
    if not path.is_file():
        print(f"Error: File '{path}' does not exist.")
        return

    print(f"Starting log analysis for: {path} ({round(path.stat().st_size / (1024**3), 2)} GB)")
    results = process_large_log(path)

    print("\n" + "=" * 50)
    print(" LOG ANALYSIS REPORT")
    print("=" * 50)
    print(f"File Name           : {results['file_name']}")
    print(f"Processed Size      : {results['file_size_gb']} GB")
    print(f"Total Lines         : {results['total_lines']:,}")
    print(f"Processing Speed    : {results['lines_per_second']:,} lines/sec ({results['execution_time_seconds']}s)")
    print("-" * 50)
    print("Log Level Counts:")
    for level, count in results["log_levels"].items():
        print(f"  - {level:<10}: {count:,}")
    
    print("\nTop Failing Modules:")
    for module, count in results["top_error_modules"]:
        print(f"  - {module:<30}: {count:,} errors")

    print("\nTop Failing Endpoints:")
    for endpoint, count in results["top_failing_endpoints"]:
        print(f"  - {endpoint:<30}: {count:,} failures")
    print("=" * 50)


if __name__ == "__main__":
    main()
```

---

## 3. Advanced Performance Comparison

| Approach | Memory Usage | Disk Usage | CPU Bottleneck | Best For |
| :--- | :--- | :--- | :--- | :--- |
| **`file.readlines()`** | High ($O(N)$ size of file) | Low | Single core | Files $< 100\text{ MB}$ |
| **Line Streaming (`for line in file`)** | **Extremely Low ($\approx 5\text{ MB}$)** | **Zero overhead** | Single core | Files $1\text{ GB} - 20\text{ GB}$ |
| **Gzip In-Memory Streaming** | **Extremely Low ($\approx 10\text{ MB}$)** | **Zero extra disk space** | Decompression bounded | Compressed logs (`.gz`) |
| **Chunked Multiprocessing (`mmap`)** | Low ($\approx 50\text{ MB}$ per core) | Zero extra disk space | Multi-core distributed | Uncompressed logs $> 50\text{ GB}$ |

---

## 4. Operational Best Practices

1. **Avoid Intermediate File Generation:** Output filtered error logs to standard output (`stdout`) or directly stream them over network connections instead of saving new files to disk.
2. **Tune Buffer Sizes:** Increasing read buffer sizes (e.g., $1\text{ MB} - 8\text{ MB}$) reduces system call overhead when reading massive files from NVMe SSDs or attached cloud volumes.
3. **Use Unix Tools for Quick Filtering:** For extremely fast pre-filtering on Linux systems prior to running Python analysis, chain `zgrep` or `rg` (ripgrep):
   ```bash
   zgrep "ERROR" production.log.gz | python3 analyze_errors.py
   ```