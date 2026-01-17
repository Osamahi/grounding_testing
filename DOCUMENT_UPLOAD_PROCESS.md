# Vertex AI Search - Document Upload Process with Chunking & Indexing

## What We Did - Step by Step (Terminal/API Level)

This document records the actual process used to upload documents to Vertex AI Search Data Store with proper chunking and indexing for grounding.

---

## Prerequisites Completed

### 1. Authentication
```bash
# Check authentication
~/google-cloud-sdk/bin/gcloud auth list

# Active account: osamah.islam@gmail.com
# Project: mojeeb (Project Number: 470233234969)
```

### 2. Enable Required API
```bash
~/google-cloud-sdk/bin/gcloud services enable discoveryengine.googleapis.com --project=mojeeb
```

### 3. Get Access Token
```bash
~/google-cloud-sdk/bin/gcloud auth print-access-token
# Returns: ya29.a0AUMWg_LU... (access token for API calls)
```

---

## Step 1: Create Vertex AI Search Data Store

### API Call
```bash
export ACCESS_TOKEN=$(~/google-cloud-sdk/bin/gcloud auth print-access-token 2>/dev/null)
export PROJECT_ID="mojeeb"
export DATASTORE_ID="mojeeb-test-datastore"

curl -X POST \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores?dataStoreId=${DATASTORE_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "displayName": "Mojeeb Test Data Store",
    "industryVertical": "GENERIC",
    "solutionTypes": ["SOLUTION_TYPE_SEARCH"],
    "contentConfig": "CONTENT_REQUIRED"
  }'
```

### Response
```json
{
  "name": "projects/470233234969/locations/global/collections/default_collection/operations/create-data-store-15461170711235673408",
  "done": true,
  "response": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.DataStore",
    "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore",
    "displayName": "Mojeeb Test Data Store",
    "industryVertical": "GENERIC",
    "solutionTypes": ["SOLUTION_TYPE_SEARCH"],
    "contentConfig": "CONTENT_REQUIRED",
    "defaultSchemaId": "default_schema",
    "documentProcessingConfig": {
      "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/documentProcessingConfig",
      "defaultParsingConfig": {
        "digitalParsingConfig": {}
      }
    },
    "servingConfigDataStore": {}
  }
}
```

### Key Details
- **Data Store ID**: `mojeeb-test-datastore`
- **Full Resource Name**: `projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore`
- **Content Config**: `CONTENT_REQUIRED` - This is CRITICAL for grounding (stores full document content)
- **Solution Type**: `SOLUTION_TYPE_SEARCH` - Required for search and grounding functionality

---

## Step 2: Upload Document to Google Cloud Storage

### Check Available Buckets
```bash
~/google-cloud-sdk/bin/gcloud storage buckets list --project=mojeeb --format="value(name)"
```

### Result
```
mojeeb_test_bucket  # ✓ Available
```

### Upload Document to GCS
```bash
~/google-cloud-sdk/bin/gcloud storage cp \
  "/Users/osamahislam/Documents/grounding testing/sample_document.txt" \
  gs://mojeeb_test_bucket/mojeeb-product-doc.txt
```

### Document Content
The uploaded document (`sample_document.txt`) contains:
- Mojeeb AI Platform product documentation
- Features: Multi-model support, grounding capabilities, Arabic language support
- Technical specifications and pricing information
- ~48 lines of text content

---

## Step 3: Create Import Manifest File (JSONL)

### Why JSONL Format?
The `importDocuments` API requires a JSONL (JSON Lines) file that specifies:
- Document IDs
- Document locations (GCS URIs)
- Metadata
- Content type

### Create JSONL File
```bash
# File: import_mojeeb_docs.jsonl
{"id":"mojeeb-product-doc","jsonData":"{}","content":{"mimeType":"text/plain","uri":"gs://mojeeb_test_bucket/mojeeb-product-doc.txt"}}
```

### JSONL Structure
```json
{
  "id": "mojeeb-product-doc",           // Unique document identifier
  "jsonData": "{}",                      // Additional metadata (empty in this case)
  "content": {
    "mimeType": "text/plain",            // Content type
    "uri": "gs://mojeeb_test_bucket/mojeeb-product-doc.txt"  // GCS URI
  }
}
```

### Upload JSONL to GCS
```bash
~/google-cloud-sdk/bin/gcloud storage cp \
  "/Users/osamahislam/Documents/grounding testing/import_mojeeb_docs.jsonl" \
  gs://mojeeb_test_bucket/import_mojeeb_docs.jsonl
```

---

## Step 4: Import Documents with Chunking & Indexing

### The Critical API Call - importDocuments

This is the CORRECT way to upload documents for grounding. This API:
- ✓ Automatically chunks documents
- ✓ Indexes content for search/retrieval
- ✓ Parses structured content
- ✓ Prepares documents for grounding

### API Endpoint
```
POST https://discoveryengine.googleapis.com/v1/projects/{project}/locations/global/collections/default_collection/dataStores/{dataStoreId}/branches/default_branch/documents:import
```

### API Call
```bash
export ACCESS_TOKEN=$(~/google-cloud-sdk/bin/gcloud auth print-access-token 2>/dev/null)
export PROJECT_ID="mojeeb"
export DATASTORE_ID="mojeeb-test-datastore"

curl -X POST \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents:import" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "gcsSource": {
      "inputUris": ["gs://mojeeb_test_bucket/import_mojeeb_docs.jsonl"],
      "dataSchema": "document"
    },
    "reconciliationMode": "INCREMENTAL"
  }'
```

### Request Body Breakdown

```json
{
  "gcsSource": {
    "inputUris": ["gs://mojeeb_test_bucket/import_mojeeb_docs.jsonl"],  // JSONL manifest
    "dataSchema": "document"  // Schema type: "document" for unstructured content
  },
  "reconciliationMode": "INCREMENTAL"  // Add new docs without deleting existing ones
}
```

#### Reconciliation Modes:
- `INCREMENTAL`: Adds or updates documents (recommended)
- `FULL`: Replaces all documents (deletes existing docs not in import)

### Response
```json
{
  "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/branches/0/operations/import-documents-16545345710860898566",
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.ImportDocumentsMetadata"
  }
}
```

### What Happens During Import?

1. **Document Retrieval**: Fetches documents from GCS URIs
2. **Parsing**: Uses default digital parser to extract content
3. **Chunking**: Automatically breaks documents into retrievable chunks
4. **Indexing**: Creates search index for efficient retrieval
5. **Metadata Storage**: Stores document metadata and structure

This is a **long-running operation** (LRO) that processes asynchronously.

---

## Step 5: Check Import Operation Status

### API Call
```bash
export OPERATION_NAME="projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/branches/0/operations/import-documents-16545345710860898566"

curl -X GET \
  "https://discoveryengine.googleapis.com/v1/${OPERATION_NAME}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}"
```

### Response (In Progress)
```json
{
  "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/branches/0/operations/import-documents-16545345710860898566",
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.ImportDocumentsMetadata",
    "createTime": "2026-01-14T15:31:38.257650Z",
    "updateTime": "2026-01-14T15:31:38.257650Z"
  }
}
```

### When Complete
The response will include:
```json
{
  "name": "...",
  "done": true,  // ✓ Operation completed
  "response": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.ImportDocumentsResponse",
    "successCount": 1,
    "failureCount": 0
  }
}
```

---

## Why This Approach is Correct for Grounding

### ❌ WRONG: Direct Document Creation
```bash
# This does NOT enable proper chunking/indexing for grounding
curl -X POST ".../documents" -d '{"content": {"rawBytes": "..."}}'
```

### ✓ CORRECT: Import Documents API
```bash
# This enables chunking, indexing, and retrieval
curl -X POST ".../documents:import" -d '{"gcsSource": {...}}'
```

### Key Differences

| Feature | Direct Upload | Import API |
|---------|--------------|------------|
| Chunking | ❌ No | ✓ Yes |
| Indexing | ❌ Basic | ✓ Full-text search index |
| Parsing | ❌ None | ✓ Digital/Layout/OCR parsers |
| Retrieval | ❌ Limited | ✓ Optimized for grounding |
| Batch Processing | ❌ One-by-one | ✓ Bulk import |

---

## Summary - The Correct Upload Flow

1. **Create Data Store** with `contentConfig: "CONTENT_REQUIRED"`
2. **Upload documents to GCS** bucket
3. **Create JSONL manifest** with document URIs
4. **Upload JSONL to GCS**
5. **Call importDocuments API** with GCS source
6. **Wait for operation completion** (check status)
7. **Use data store for grounding** in Gemini API calls

---

## Next Steps: Using for Grounding

Once documents are imported and indexed, reference the data store in Vertex AI API calls:

```json
{
  "contents": [{
    "role": "user",
    "parts": [{"text": "What is Mojeeb?"}]
  }],
  "tools": [{
    "retrieval": {
      "vertexAiSearch": {
        "datastore": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore"
      }
    }
  }]
}
```

This enables **true grounding** - the model retrieves relevant chunks from your documents and grounds its responses in that content.

---

## Important Notes

1. **Import is asynchronous** - Wait 1-5 minutes for documents to be fully indexed
2. **contentConfig must be CONTENT_REQUIRED** - Otherwise grounding won't work
3. **Use JSONL format** - Not individual JSON objects
4. **GCS URIs required** - Cannot upload directly from local filesystem
5. **Data schema is "document"** - For unstructured content (text, PDFs, etc.)

---

## Files Created

- `sample_document.txt` - Source document with Mojeeb product info
- `import_mojeeb_docs.jsonl` - Import manifest
- GCS: `gs://mojeeb_test_bucket/mojeeb-product-doc.txt`
- GCS: `gs://mojeeb_test_bucket/import_mojeeb_docs.jsonl`

## API Resources Created

- Data Store: `projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore`
- Import Operation: `import-documents-16545345710860898566`
