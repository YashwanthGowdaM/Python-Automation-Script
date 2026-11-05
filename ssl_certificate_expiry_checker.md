# SSL Certificate Expiry Checker

This script inspects multiple application URLs over TLS/SSL, parses their SSL certificates, calculates the remaining days before expiration, and highlights certificates expiring within 30 days.

## Features
- Extracts hostnames directly from full URLs.
- Calculates exact days remaining until expiry.
- Alerts with clear warnings for certificates nearing expiration or already expired.

## Python Implementation (`ssl_checker.py`)

```python
import socket
import ssl
from datetime import datetime
from urllib.parse import urlparse
from typing import List, Dict, Any, Tuple

# Threshold for warning in days
WARNING_THRESHOLD_DAYS = 30


def extract_hostname_and_port(url: str) -> Tuple[str, int]:
    """Parses URL to extract the hostname and port."""
    if not url.startswith("http://") and not url.startswith("https://"):
        url = "https://" + url  # Default to HTTPS for parsing

    parsed = urlparse(url)
    hostname = parsed.hostname or url
    port = parsed.port if parsed.port else 443
    return hostname, port


def check_ssl_expiry(url: str) -> Dict[str, Any]:
    """Connects to server over SSL and retrieves certificate expiration date."""
    hostname, port = extract_hostname_and_port(url)
    context = ssl.create_default_context()

    try:
        with socket.create_connection((hostname, port), timeout=5.0) as sock:
            with context.wrap_socket(sock, server_hostname=hostname) as ssock:
                cert = ssock.getpeercert()

                # Expiry string format e.g., 'May 20 12:00:00 2026 GMT'
                expiry_str = cert.get("notAfter")
                expiry_date = datetime.strptime(expiry_str, "%b %d %H:%M:%S %Y %Z")
                days_left = (expiry_date - datetime.utcnow()).days

                return {
                    "url": url,
                    "hostname": hostname,
                    "expiry_date": expiry_date.strftime("%Y-%m-%d %H:%M:%S UTC"),
                    "days_remaining": days_left,
                    "status": "WARNING" if days_left <= WARNING_THRESHOLD_DAYS else "OK",
                    "error": None
                }

    except Exception as e:
        return {
            "url": url,
            "hostname": hostname,
            "expiry_date": None,
            "days_remaining": -1,
            "status": "ERROR",
            "error": str(e)
        }


def run_ssl_check(urls: List[str]):
    """Executes SSL check across a list of URLs and displays report."""
    print("=" * 80)
    print(f"{'URL / HOSTNAME':<35} | {'STATUS':<8} | {'DAYS LEFT':<10} | {'EXPIRY DATE'}")
    print("=" * 80)

    for url in urls:
        res = check_ssl_expiry(url)
        if res["status"] == "ERROR":
            print(f"{res['hostname']:<35} | {'ERROR':<8} | {'N/A':<10} | {res['error']}")
        else:
            status_flag = "⚠️ WARN" if res["status"] == "WARNING" else "✅ OK"
            print(f"{res['hostname']:<35} | {status_flag:<8} | {res['days_remaining']:<10} | {res['expiry_date']}")

    print("=" * 80)


if __name__ == "__main__":
    target_urls = [
        "https://google.com",
        "https://github.com",
        "https://expired.badssl.com",
        "https://amazon.com"
    ]
    run_ssl_check(target_urls)
```