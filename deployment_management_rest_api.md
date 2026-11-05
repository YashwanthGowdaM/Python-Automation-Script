# Deployment Management REST API

A Flask RESTful service built to manage, trigger, and track application deployments across environments.

## Endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| `POST` | `/api/v1/deployments` | Submit a deployment request (`app_name`, `environment`, `version`). |
| `GET` | `/api/v1/deployments` | Retrieve all registered deployment logs. |
| `GET` | `/api/v1/deployments/<id>` | Fetch deployment details by ID. |

## Python Implementation (`app.py`)

```python
import uuid
from datetime import datetime
from typing import Dict, Any
from flask import Flask, request, jsonify

app = Flask(__name__)

# Allowed environments validation set
VALID_ENVIRONMENTS = {"dev", "staging", "production"}

# In-memory deployment database
deployments_db: Dict[str, Dict[str, Any]] = {}


@app.route("/api/v1/deployments", methods=["POST"])
def create_deployment():
    """
    Submits a new deployment request.
    Expected JSON payload:
    {
        "app_name": "payment-service",
        "environment": "production",
        "version": "v2.1.0"
    }
    """
    data = request.get_json()

    if not data or not isinstance(data, dict):
        return jsonify({"status": "error", "message": "Invalid or missing JSON payload"}), 400

    app_name = data.get("app_name")
    environment = data.get("environment")
    version = data.get("version")

    # Input Validations
    errors = []
    if not app_name or not isinstance(app_name, str):
        errors.append("Field 'app_name' is required and must be a string.")
    if not environment or environment.lower() not in VALID_ENVIRONMENTS:
        errors.append(f"Field 'environment' must be one of {list(VALID_ENVIRONMENTS)}.")
    if not version or not isinstance(version, str):
        errors.append("Field 'version' is required and must be a string.")

    if errors:
        return jsonify({"status": "error", "validation_errors": errors}), 422

    # Create Deployment Record
    deployment_id = str(uuid.uuid4())
    record = {
        "id": deployment_id,
        "app_name": app_name.strip(),
        "environment": environment.lower().strip(),
        "version": version.strip(),
        "status": "QUEUED",
        "created_at": datetime.utcnow().isoformat() + "Z"
    }

    deployments_db[deployment_id] = record

    return jsonify({
        "status": "success",
        "message": "Deployment queued successfully.",
        "deployment": record
    }), 201


@app.route("/api/v1/deployments", methods=["GET"])
def list_deployments():
    """Retrieves all deployment records."""
    return jsonify({
        "status": "success",
        "count": len(deployments_db),
        "deployments": list(deployments_db.values())
    }), 200


@app.route("/api/v1/deployments/<string:deployment_id>", methods=["GET"])
def get_deployment(deployment_id: str):
    """Retrieves deployment record by ID."""
    record = deployments_db.get(deployment_id)
    if not record:
        return jsonify({"status": "error", "message": "Deployment ID not found"}), 404

    return jsonify({
        "status": "success",
        "deployment": record
    }), 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```