# AWS EC2 Resource Inventory Reporter

This Python script utilizes `boto3` to inspect EC2 instances within a specified AWS region, extract details including instance name, type, state, and private IP address, and save the aggregated data into a clean CSV report.

## Features
- Handles missing tags gracefully (e.g., missing `Name` tag fallback).
- Catches AWS API errors (`ClientError`, `NoCredentialsError`, `BotoCoreError`).
- Configurable region and output filename.

## Python Implementation (`ec2_inventory.py`)

```python
import csv
import logging
import sys
from typing import List, Dict, Any
import boto3
from botocore.exceptions import ClientError, NoCredentialsError, BotoCoreError

# Configure Logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)]
)
logger = logging.getLogger(__name__)


def extract_instance_name(tags: List[Dict[str, str]]) -> str:
    """Extracts the 'Name' tag value from an EC2 instance's tags list."""
    if not tags:
        return "N/A"
    for tag in tags:
        if tag.get("Key") == "Name":
            return tag.get("Value", "N/A")
    return "N/A"


def fetch_ec2_inventory(region: str) -> List[Dict[str, Any]]:
    """
    Fetches EC2 instances details across a specified AWS region.
    """
    logger.info(f"Connecting to AWS EC2 in region: {region}...")
    inventory = []

    try:
        ec2_client = boto3.client("ec2", region_name=region)
        paginator = ec2_client.get_paginator("describe_instances")

        for page in paginator.paginate():
            for reservation in page.get("Reservations", []):
                for instance in reservation.get("Instances", []):
                    instance_id = instance.get("InstanceId", "Unknown")
                    instance_type = instance.get("InstanceType", "Unknown")
                    state = instance.get("State", {}).get("Name", "Unknown")
                    private_ip = instance.get("PrivateIpAddress", "N/A")
                    tags = instance.get("Tags", [])
                    name = extract_instance_name(tags)

                    inventory.append({
                        "InstanceID": instance_id,
                        "Name": name,
                        "Status": state,
                        "InstanceType": instance_type,
                        "PrivateIP": private_ip,
                        "Region": region
                    })

    except NoCredentialsError:
        logger.error("AWS credentials not found. Please configure your AWS profile or environment variables.")
        raise
    except ClientError as ce:
        logger.error(f"AWS API Error encountered: {ce.response['Error']['Message']}")
        raise
    except BotoCoreError as bce:
        logger.error(f"BotoCore error occurred: {bce}")
        raise

    return inventory


def save_inventory_to_csv(inventory: List[Dict[str, Any]], output_file: str) -> None:
    """Saves the extracted instance inventory to a CSV file."""
    fieldnames = ["InstanceID", "Name", "Status", "InstanceType", "PrivateIP", "Region"]

    try:
        with open(output_file, mode="w", newline="", encoding="utf-8") as file:
            writer = csv.DictWriter(file, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(inventory)
        logger.info(f"Successfully generated inventory report: {output_file}")
    except IOError as ioe:
        logger.error(f"Failed to write CSV file to {output_file}: {ioe}")
        raise


def main():
    # Parameters
    aws_region = "us-east-1"
    report_filename = "ec2_inventory_report.csv"

    try:
        instances = fetch_ec2_inventory(region=aws_region)
        logger.info(f"Found {len(instances)} EC2 instance(s).")
        save_inventory_to_csv(instances, output_file=report_filename)
    except Exception as e:
        logger.critical(f"Execution failed: {e}")
        sys.exit(1)


if __name__ == "__main__":
    main()
```