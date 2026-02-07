# Architectural Design Guide: Drug-Drug Interaction (DDI) Checker System

![Type](https://img.shields.io/badge/Type-Technical_Guide-blue) ![Domain](https://img.shields.io/badge/Domain-Healthcare_IT-red) ![API](https://img.shields.io/badge/Integration-DrugBank_API-green)

> **Note:** This guide bridges the gap between pharmacological theory and technical implementation, focusing on clinical safety and data integrity standards suitable for CDSS (Clinical Decision Support Systems).

## 📋 Table of Contents
1. [Introduction & Clinical Relevance](#1-introduction--clinical-relevance)
2. [API Architecture & Data Standards](#2-api-architecture--data-standards)
3. [Security & Authentication](#3-security--authentication)
4. [Implementation Guide (DDI Checker)](#4-implementation-guide-ddi-checker)
5. [Code Implementation](#5-code-implementation)

---

## 1. Introduction & Clinical Relevance

In the era of eHealth, Drug-Drug Interaction (DDI) screening is critical for preventing adverse drug events, especially in patients with polypharmacy.

This guide details the architectural integration of the **DrugBank Clinical API** to facilitate:
* **Real-time DDI Screening:** Detecting conflicts between prescribed substances dynamically.
* **Clinical Accuracy:** Moving away from static datasets to real-time, curated pharmacological data.
* **Regulatory Compliance:** Ensuring data aligns with regional standards (FDA, EMA).

---

## 2. API Architecture & Data Standards

The system utilizes a **RESTful architecture**, leveraging standard HTTP verbs for communication.

### Data Formats: JSON vs. XML
* **Live REST API (JSON):** Optimized for real-time web applications. Requires the `Accept: application/json` header.
* **Legacy Data (XML):** Full database exports are available in XML for bulk processing and interoperability with legacy Healthcare Information Systems (HIS) and HL7 standards.

### Regional Compliance
To ensure results match local regulatory approvals (FDA vs EMA), developers must target the correct region. While legacy integrations used region-specific base URLs, the modern standard utilizes query parameters or API key settings.

* **Standard Method:** Append the `region` parameter to your request.
  * 🇺🇸 US: `?region=us` (Default)
  * 🇪🇺 EU: `?region=eu`
  * 🇨🇦 CA: `?region=ca`

> **Note:** The API Base URL is not a browsable website. It is an endpoint accessed programmatically via code (cURL, Python, etc.).

---

## 3. Security & Authentication

Healthcare data security is non-negotiable. All API communication requires **HTTPS** (TLS) encryption.

### Authentication Strategies
| Method | Description | Use Case |
| :--- | :--- | :--- |
| **API Key** | Secret alphanumeric string | Strictly for **Server-side** (Backend) logic. |
| **Client Token** | Temporary, time-limited token | Suitable for **Client-side** (Mobile/Web Apps). |

> ⚠️ **Security Warning:** Never commit hard-coded API Keys to version control. Use Environment Variables (`.env`) or Secret Managers.

---

## 4. Implementation Guide (DDI Checker)

### Identifiers
Precise drug identification is achieved using standard vocabularies:
* `DrugBank ID` (e.g., DB00316 - Acetaminophen)
* `NDC` (National Drug Code - FDA standard)
* `RxCUI` (RxNorm Concept Unique Identifier)

### Core Endpoints
* **Single Drug Check (GET):** `/v1/drugs/{id}/ddi`
* **Multi-Drug Interaction Checker (POST):** `/v1/ddi`
    * *Usage:* Validates a list of drugs simultaneously. The request body accepts complex JSON structures containing multiple identifiers.

---

## 5. Code Implementation

Below are production-ready examples for integrating the DDI Checker.

### 🐍 Python Implementation
Uses the `requests` library with robust error handling and rate limit management.

```python
import requests
import json
import os

# Best Practice: Load credentials from Environment Variables
API_KEY = os.getenv("DRUGBANK_API_KEY") 

# Base URL for the DDI endpoint
BASE_URL = "[https://api.drugbank.com/v1/ddi](https://api.drugbank.com/v1/ddi)"

def check_interactions(drug_ids, region='us'):
    """
    Checks for interactions between a list of drugs.
    :param drug_ids: List of DrugBank IDs
    :param region: Regulatory region (us, eu, ca)
    """
    headers = {
        "Authorization": API_KEY,
        "Content-Type": "application/json",
        "Accept": "application/json"
    }
    
    # Payload construction
    payload = { "drugbank_id": drug_ids }
    
    # Append region parameter
    url = f"{BASE_URL}?region={region}"

    try:
        response = requests.post(url, headers=headers, json=payload)
        
        # Check for HTTP errors
        response.raise_for_status()
        
        data = response.json()
        print(f"Total interactions found ({region.upper()}): {data.get('total_results', 0)}")

        for interaction in data.get('interactions', []):
            drug_a = interaction['product_ingredient']['name']
            drug_b = interaction['affected_product_ingredient']['name']
            severity = interaction['severity']
            description = interaction['description']
            
            # Log interaction with severity level
            print(f"[{severity.upper()}] {drug_a} <--> {drug_b}: {description}")

    except requests.exceptions.HTTPError as err:
        if response.status_code == 429:
            print("Error: Rate limit exceeded. Implementing backoff strategy...")
        elif response.status_code == 401:
            print("Error: Unauthorized. Check your API Key.")
        else:
            print(f"HTTP Error: {err}")
    except Exception as e:
        print(f"General Error: {e}")

if __name__ == "__main__":
    # Example: Acetaminophen, Abatacept, Lepirudin
    drugs_to_check = ["DB00316", "DB01281", "DB00001"]
    # Check against EU regulations
    check_interactions(drugs_to_check, region='eu')
