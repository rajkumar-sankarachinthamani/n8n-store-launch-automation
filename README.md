# n8n Store Launch Automation Report

An n8n workflow that automates weekly **Store Launch** reporting by querying MongoDB, formatting results with AI, and delivering a polished HTML email via Gmail.

---

## Overview

This workflow runs on a **weekly schedule** and performs the following end-to-end pipeline:

1. **Triggers** automatically every Monday at 6:10 AM.
2. Uses an **AI Agent** (GPT-4o-mini) to generate a MongoDB aggregation query from a natural-language prompt.
3. **Executes** the generated aggregation pipeline against MongoDB to join `TaggingSummary` and `Location` collections.
4. **Transforms** and formats the query results into a structured table with site counts.
5. Uses **OpenAI** (GPT-4o-mini) to generate a clean, professional HTML email from the data.
6. **Sends** the final report via **Gmail** (OAuth2) to the designated recipient.

---

## Workflow Architecture

```
Schedule Trigger (Mon 6:10 AM)
       |
       v
   AI Agent (GPT-4o-mini)
   +-- System Prompt: NL -> MongoDB query converter
   +-- Output: MongoDB aggregation pipeline JSON
       |
       v
   Code: Parse AI Output
   +-- Strips code fences, validates JSON
   +-- Extracts collection name + pipeline array
       |
       v
   MongoDB: Aggregate Documents
   +-- Executes $lookup join: TaggingSummary <-> Location
   +-- Join keys: siteEPC <-> location_epc
   +-- Returns: siteNumber, location_description, min(date)
       |
       v
   Code: Combine Results
   +-- Merges all MongoDB documents into a single array
       |
       v
   Code: Format Table + Count
   +-- Builds tab-separated table from results
   +-- Calculates total site count
       |
       v
   OpenAI: Format HTML Email (GPT-4o-mini)
   +-- Converts table data into professional HTML
   +-- Includes greeting, summary stats, and signature
       |
       v
   Gmail: Send Report (OAuth2)
   +-- Delivers the HTML email to the recipient
```

---

## Node Details

### 1. Schedule Trigger
- **Type:** `n8n-nodes-base.scheduleTrigger`
- **Schedule:** Every **Monday** at **6:10 AM**
- **Purpose:** Initiates the weekly reporting pipeline automatically.

### 2. AI Agent
- **Type:** `@n8n/n8n-nodes-langchain.agent`
- **Model:** GPT-4o-mini (via OpenAI)
- **System Prompt Behavior:**
  - Converts natural-language questions into valid MongoDB aggregation pipelines.
  - Uses `$lookup` for collection joins, `$unwind` for arrays.
  - Converts numeric timestamps to dates using `$toDate`.
  - Outputs strict JSON -- no SQL, no comments, no hallucination.
- **Query:** Joins `TaggingSummary` and `Location` on `siteEPC` <-> `location_epc`, returning `siteNumber`, `location_description`, and the minimum `date`.

### 3. Code: Parse AI Output
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:** Cleans and parses the AI Agent's string output.
  - Removes markdown code fences.
  - Validates JSON structure, ensures `pipeline` is an array.
  - Extracts `Collection` and `pipeline` for the MongoDB node.

### 4. MongoDB: Aggregate Documents
- **Type:** `n8n-nodes-base.mongoDb`
- **Operation:** `aggregate`
- **Details:**
  - Runs the AI-generated aggregation pipeline.
  - Performs a `$lookup` join between `TaggingSummary` and `Location`.
  - Returns matched documents with site information and earliest dates.

### 5. Code: Combine Results
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:** Merges all individual MongoDB result items into a single `data` array for downstream processing.

### 6. Code: Format Table + Site Count
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:**
  - Dynamically extracts column headers from result keys.
  - Builds a tab-separated table string.
  - Calculates the total **site count** from the data rows.

### 7. OpenAI: Format HTML Email
- **Type:** `@n8n/n8n-nodes-langchain.openAi`
- **Model:** GPT-4o-mini
- **Configuration:**
  - Temperature: `0.1`, Top-P: `0.1` (deterministic output).
  - Generates a professional HTML email with:
    - Greeting and summary header.
    - Proper HTML `<table>` with headers and data rows.
    - Uses the **exact site count** provided (no recalculation).
    - Professional closing signature.

### 8. Gmail: Send Report
- **Type:** `n8n-nodes-base.gmail`
- **Authentication:** Gmail OAuth2
- **Details:**
  - Subject: `Store Launch Weekly report`
  - Sends the AI-formatted HTML email body to the configured recipient.

---

## Tech Stack

| Component | Technology |
|---|---|
| Workflow Engine | n8n (self-hosted or cloud) |
| AI / LLM | OpenAI GPT-4o-mini |
| Database | MongoDB (aggregation pipelines) |
| Email Delivery | Gmail (OAuth2) |
| Scripting | JavaScript (n8n Code nodes) |

---

## Repository Structure

```
n8n-store-launch-automation/
+-- README.md                  # This file
+-- store_extract.json         # n8n workflow (import directly into n8n)
+-- .gitignore                 # Git ignore rules
```

---

## Setup and Usage

### Prerequisites
- **n8n** instance (self-hosted or n8n Cloud)
- **OpenAI API key** with access to `gpt-4o-mini`
- **MongoDB** connection with `TaggingSummary` and `Location` collections
- **Gmail OAuth2** credentials configured in n8n

### Import the Workflow
1. Open your n8n instance.
2. Go to **Workflows -> Import from File**.
3. Select `store_extract.json`.
4. Configure your credentials:
   - **OpenAI API** -- Add your API key.
   - **MongoDB** -- Set your connection string and database.
   - **Gmail OAuth2** -- Authorize your Google account.
5. Update the recipient email in the **Gmail node** if needed.
6. **Activate** the workflow.

---

## Configuration Notes

- **Schedule:** Default is Monday 6:10 AM. Adjust in the Schedule Trigger node.
- **Recipient:** Update `sendTo` in the Gmail node to change the email recipient.
- **AI Prompt:** Modify the AI Agent's `text` parameter to change the MongoDB query logic.
- **Email Template:** Customize the greeting and signature in the OpenAI node's prompt.

---

## Security Considerations

> **Important:** This workflow file contains **credential reference IDs** (not secrets). You must configure your own credentials in n8n after import. Never commit actual API keys or passwords.

##n8n Workflow Screen shot
<img width="1582" height="542" alt="image" src="https://github.com/user-attachments/assets/5479a056-ccfa-4478-a85c-c94753bfc249" />

---

## License

This project is provided as-is for internal automation use.
