# Multi-Server Deployment Verification Script

This script uses SSH (`paramiko`) to concurrently verify application deployment status, service health, and software version across multiple target servers.

## Features
- Parallel connection handling using standard Python `concurrent.futures`.
- Runs remote health-check commands (e.g., verifying `systemctl` service status, application version checks).
- Gracefully handles SSH timeouts, invalid credentials, and connection failures.

## Python Implementation (`verify_deployment.py`)

```python
import logging
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Dict, Any, List
import paramiko

# Configure Logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")

# Server inventory configuration
SERVERS = [
    {"host": "192.168.1.10", "username": "admin", "key_path": "~/.ssh/id_rsa"},
    {"host": "192.168.1.11", "username": "admin", "key_path": "~/.ssh/id_rsa"},
    {"host": "192.168.1.12", "username": "admin", "key_path": "~/.ssh/id_rsa"}
]

# Command to execute on remote servers
CHECK_COMMAND = "systemctl is-active app-service && cat /opt/app/version.txt"


def verify_server_deployment(server: Dict[str, str], command: str) -> Dict[str, Any]:
    """Connects to a single server via SSH and executes verification commands."""
    host = server["host"]
    username = server["username"]
    key_path = server["key_path"]

    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    try:
        logging.info(f"Connecting to {host}...")
        ssh.connect(
            hostname=host,
            username=username,
            key_filename=key_path,
            timeout=5.0
        )

        stdin, stdout, stderr = ssh.exec_command(command, timeout=5.0)
        exit_status = stdout.channel.recv_exit_status()

        out_msg = stdout.read().decode().strip()
        err_msg = stderr.read().decode().strip()

        if exit_status == 0:
            return {
                "host": host,
                "status": "SUCCESS",
                "output": out_msg,
                "error": None
            }
        else:
            return {
                "host": host,
                "status": "FAILED",
                "output": out_msg,
                "error": err_msg or f"Command exited with status code {exit_status}"
            }

    except Exception as e:
        return {
            "host": host,
            "status": "UNREACHABLE",
            "output": "",
            "error": str(e)
        }
    finally:
        ssh.close()


def run_deployment_verification(servers: List[Dict[str, str]], command: str):
    """Orchestrates parallel verification across all servers using thread pools."""
    logging.info("Starting Multi-Server Deployment Verification...")
    results = []

    with ThreadPoolExecutor(max_workers=5) as executor:
        future_to_host = {
            executor.submit(verify_server_deployment, server, command): server["host"]
            for server in servers
        }

        for future in as_completed(future_to_host):
            res = future.result()
            results.append(res)

    # Display Report
    print("\n" + "=" * 70)
    print(" DEPLOYMENT VERIFICATION REPORT")
    print("=" * 70)
    for res in results:
        status_symbol = "✅" if res["status"] == "SUCCESS" else "❌"
        print(f"Host: {res['host']:<15} | Status: {status_symbol} {res['status']}")
        if res["output"]:
            print(f"  Output : {res['output']}")
        if res["error"]:
            print(f"  Error  : {res['error']}")
        print("-" * 70)


if __name__ == "__main__":
    run_deployment_verification(SERVERS, CHECK_COMMAND)
```