# Vertex AI Search & Grounding - Complete API Reference

## Overview
Complete REST API documentation for Vertex AI Search and Grounding with Gemini. This document covers the entire workflow from data store creation to grounded chat responses.

**Tested with:**
- Project: mojeeb (470233234969)
- Model: gemini-2.5-flash
- Date: 2026-01-14

---

## Table of Contents

1. [Workflow Overview](#workflow-overview)
2. [Prerequisites](#prerequisites)
3. [API 1: Create Data Store](#api-1-create-data-store)
4. [API 2: List Data Stores](#api-2-list-data-stores)
5. [API 3: Upload Documents to GCS](#api-3-upload-documents-to-gcs)
6. [API 4: Import Documents](#api-4-import-documents)
7. [API 5: Check Indexing Status](#api-5-check-indexing-status)
8. [API 6: List Documents](#api-6-list-documents)
9. [API 7: Get Single Document](#api-7-get-single-document)
10. [API 8: Delete Document](#api-8-delete-document)
11. [API 9: Grounded Chat with Gemini](#api-9-grounded-chat-with-gemini)
12. [Complete Request/Response Models](#complete-requestresponse-models)
13. [Error Handling](#error-handling)
14. [End-to-End Example](#end-to-end-example)
15. [References](#references)

---

## Workflow Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Vertex AI Grounding Workflow               │
└─────────────────────────────────────────────────────────────┘

Step 1: Create Data Store
  ↓
  POST /v1/projects/{project}/locations/global/collections/default_collection/dataStores
  → Returns: Data Store resource name

Step 2: Upload Documents to GCS
  ↓
  POST https://storage.googleapis.com/upload/storage/v1/b/{bucket}/o
  → Uploads: Document file to Cloud Storage

Step 3: Create JSONL Manifest
  ↓
  Create: manifest.jsonl with document URIs
  Upload: manifest.jsonl to GCS

Step 4: Import Documents to Data Store
  ↓
  POST /v1/.../dataStores/{id}/branches/default_branch/documents:import
  → Triggers: Chunking, parsing, and indexing
  → Returns: Long-running operation

Step 5: Check Indexing Status
  ↓
  GET /v1/{operation_name}
  → Wait until: done: true, successCount > 0

Step 6: Chat with Grounding
  ↓
  POST /v1beta1/.../models/gemini-2.5-flash:generateContent
  → Returns: Grounded response with source attribution
```

### Key Points
- **Chunking**: Automatic when using importDocuments API
- **Indexing**: Takes 1-3 minutes after import
- **Grounding**: Requires `contentConfig: "CONTENT_REQUIRED"`
- **API Version**: Use `v1beta1` for Gemini grounding

---

## Prerequisites

### 1. Enable Required APIs
```bash
gcloud services enable discoveryengine.googleapis.com \
  aiplatform.googleapis.com \
  storage.googleapis.com \
  --project=YOUR_PROJECT_ID
```

### 2. Authentication
```bash
# Get access token (expires in 1 hour)
gcloud auth print-access-token

# For application default credentials
gcloud auth application-default login --project=YOUR_PROJECT_ID
gcloud auth application-default set-quota-project YOUR_PROJECT_ID
```

### 3. Create GCS Bucket
```bash
gsutil mb -p YOUR_PROJECT_ID gs://YOUR_BUCKET_NAME
```

---

## API 1: Create Data Store

Creates a Vertex AI Search data store for document storage and retrieval.

### Endpoint
```
POST https://discoveryengine.googleapis.com/v1/projects/{project}/locations/{location}/collections/{collection}/dataStores
```

### URL Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{project}` | Project ID or number | `mojeeb` or `470233234969` |
| `{location}` | Data store location | `global` |
| `{collection}` | Collection name | `default_collection` |

### Query Parameters
| Parameter | Required | Description |
|-----------|----------|-------------|
| `dataStoreId` | Yes | Unique identifier (lowercase, alphanumeric, hyphens) |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json
X-Goog-User-Project: {PROJECT_ID}
```

### Request Body
```json
{
  "displayName": "string",
  "industryVertical": "GENERIC",
  "solutionTypes": ["SOLUTION_TYPE_CHAT"],
  "contentConfig": "CONTENT_REQUIRED"
}
```

#### Request Fields
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `displayName` | string | Yes | Human-readable name for the data store |
| `industryVertical` | enum | Yes | Industry vertical (use `GENERIC` for general purpose) |
| `solutionTypes` | array | Yes | Use `["SOLUTION_TYPE_CHAT"]` for chat grounding |
| `contentConfig` | enum | Yes | Use `CONTENT_REQUIRED` for document-based grounding |

**Note:** This configuration is specifically for **document-based grounding**. The `CONTENT_REQUIRED` setting means you will upload your own documents (PDFs, TXTs, etc.) for the AI to reference.

### Response (200 OK)
```json
{
  "name": "projects/470233234969/locations/global/collections/default_collection/operations/create-data-store-15931574643748153838",
  "done": true,
  "response": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.DataStore",
    "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/test-datastore-api",
    "displayName": "Test DataStore via API",
    "industryVertical": "GENERIC",
    "solutionTypes": ["SOLUTION_TYPE_CHAT"],
    "contentConfig": "CONTENT_REQUIRED",
    "defaultSchemaId": "default_schema",
    "advancedSiteSearchConfig": {
      "disableAutomaticRefresh": false
    },
    "documentProcessingConfig": {
      "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/test-datastore-api/documentProcessingConfig",
      "defaultParsingConfig": {
        "digitalParsingConfig": {}
      }
    },
    "servingConfigDataStore": {}
  }
}
```

#### Response Fields
| Field | Description |
|-------|-------------|
| `name` | Operation resource name |
| `done` | Boolean, true if complete |
| `response.name` | **Full data store resource name** (save this!) |
| `response.documentProcessingConfig` | Default parsing configuration |

### Example Command
```bash
# Get fresh access token and save to file
gcloud auth print-access-token > /tmp/token.txt

# Create data store
curl -X POST "https://discoveryengine.googleapis.com/v1/projects/mojeeb/locations/global/collections/default_collection/dataStores?dataStoreId=my-datastore-id" \
  -H "Authorization: Bearer $(cat /tmp/token.txt | tr -d '\n')" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: mojeeb" \
  -d '{
    "displayName": "My DataStore for Grounding",
    "industryVertical": "GENERIC",
    "solutionTypes": ["SOLUTION_TYPE_CHAT"],
    "contentConfig": "CONTENT_REQUIRED"
  }'
```

---

## API 2: List Data Stores

Retrieves all data stores in a project for selection and management.

### Endpoint
```
GET https://discoveryengine.googleapis.com/v1/projects/{project}/locations/{location}/collections/{collection}/dataStores
```

### URL Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{project}` | Project ID or number | `mojeeb` or `470233234969` |
| `{location}` | Data store location | `global` |
| `{collection}` | Collection name | `default_collection` |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
X-Goog-User-Project: {PROJECT_ID}
```

### Response (200 OK)
```json
{
  "dataStores": [
    {
      "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/agent1",
      "displayName": "agent1",
      "industryVertical": "GENERIC",
      "createTime": "2026-01-14T16:15:31.812785Z",
      "solutionTypes": ["SOLUTION_TYPE_CHAT"],
      "contentConfig": "CONTENT_REQUIRED",
      "defaultSchemaId": "default_schema",
      "advancedSiteSearchConfig": {
        "disableAutomaticRefresh": false
      },
      "documentProcessingConfig": {
        "name": "projects/470233234969/.../documentProcessingConfig",
        "defaultParsingConfig": {
          "digitalParsingConfig": {}
        }
      },
      "servingConfigDataStore": {}
    }
  ]
}
```

#### Response Fields
| Field | Description |
|-------|-------------|
| `dataStores` | Array of data store objects |
| `dataStores[].name` | Full resource name of the data store |
| `dataStores[].displayName` | Human-readable name |
| `dataStores[].industryVertical` | Industry vertical type |
| `dataStores[].createTime` | ISO 8601 creation timestamp |
| `dataStores[].solutionTypes` | Array of solution types |
| `dataStores[].contentConfig` | Content configuration type |

### Example Command
```bash
# Get fresh access token
gcloud auth print-access-token > /tmp/token.txt

# List all data stores
curl -X GET "https://discoveryengine.googleapis.com/v1/projects/mojeeb/locations/global/collections/default_collection/dataStores" \
  -H "Authorization: Bearer $(cat /tmp/token.txt | tr -d '\n')" \
  -H "X-Goog-User-Project: mojeeb"
```

### Use Case
This API is essential for:
- Populating dropdown menus for data store selection
- Discovering available data stores in a project
- Checking data store configuration and status
- Building management UIs

### Extracting Data Store ID
```javascript
// JavaScript example
const dataStores = response.dataStores || [];
dataStores.forEach(ds => {
  const datastoreId = ds.name.split('/').pop(); // Extract last part
  const displayName = ds.displayName || datastoreId;
  console.log(`${displayName} (${datastoreId})`);
});
```

---

## API 3: Upload Documents to GCS

Uploads document files to Google Cloud Storage before importing to Vertex AI.

### Endpoint
```
POST https://storage.googleapis.com/upload/storage/v1/b/{bucket}/o?uploadType=media&name={objectName}
```

### URL Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{bucket}` | GCS bucket name | `mojeeb_test_bucket` |
| `{objectName}` | File path in bucket | `documents/doc.pdf` |

### Query Parameters
| Parameter | Required | Description |
|-----------|----------|-------------|
| `uploadType` | Yes | Use `media` for simple upload |
| `name` | Yes | Object name/path in bucket |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: {MIME_TYPE}
Content-Length: {FILE_SIZE}
```

### Request Body
Binary file content

### Response (200 OK)
```json
{
  "kind": "storage#object",
  "id": "mojeeb_test_bucket/mojeeb-product-doc.txt/1705243890123456",
  "selfLink": "https://www.googleapis.com/storage/v1/b/mojeeb_test_bucket/o/mojeeb-product-doc.txt",
  "mediaLink": "https://storage.googleapis.com/download/storage/v1/b/mojeeb_test_bucket/o/mojeeb-product-doc.txt?generation=1705243890123456&alt=media",
  "name": "mojeeb-product-doc.txt",
  "bucket": "mojeeb_test_bucket",
  "generation": "1705243890123456",
  "metageneration": "1",
  "contentType": "text/plain",
  "storageClass": "STANDARD",
  "size": "1887",
  "md5Hash": "abc123...",
  "crc32c": "xyz789...",
  "etag": "CNvQ...",
  "timeCreated": "2026-01-14T15:24:50.123Z",
  "updated": "2026-01-14T15:24:50.123Z",
  "timeStorageClassUpdated": "2026-01-14T15:24:50.123Z"
}
```

#### Response Fields
| Field | Description |
|-------|-------------|
| `name` | Object name in bucket |
| `bucket` | Bucket name |
| `size` | File size in bytes |
| `contentType` | MIME type |
| `mediaLink` | Download URL |
| `timeCreated` | Upload timestamp |

### Supported File Types
| Type | Extensions | Max Size |
|------|-----------|----------|
| Text | .txt | 200 MB |
| PDF | .pdf | 200 MB |
| Word | .doc, .docx | 200 MB |
| PowerPoint | .ppt, .pptx | 200 MB |
| Excel | .xls, .xlsx, .xlsm | 200 MB |
| HTML | .html | 200 MB |

### Example Command
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
export BUCKET="mojeeb_test_bucket"
export FILE_PATH="/path/to/document.txt"
export FILE_NAME="mojeeb-product-doc.txt"

curl -X POST \
  "https://storage.googleapis.com/upload/storage/v1/b/${BUCKET}/o?uploadType=media&name=${FILE_NAME}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: text/plain" \
  --data-binary "@${FILE_PATH}"
```

### Using gcloud
```bash
gcloud storage cp "/path/to/document.txt" gs://mojeeb_test_bucket/mojeeb-product-doc.txt
```

---

## API 4: Import Documents

Imports documents from GCS to Vertex AI Search with automatic chunking and indexing.

### Endpoint
```
POST https://discoveryengine.googleapis.com/v1/projects/{project}/locations/global/collections/default_collection/dataStores/{datastoreId}/branches/default_branch/documents:import
```

### URL Parameters
| Parameter | Description |
|-----------|-------------|
| `{project}` | Project ID |
| `{datastoreId}` | Data store ID |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json
X-Goog-User-Project: {PROJECT_ID}
```

### Request Body
```json
{
  "gcsSource": {
    "inputUris": ["gs://bucket/manifest.jsonl"],
    "dataSchema": "document"
  },
  "reconciliationMode": "INCREMENTAL"
}
```

#### Request Fields
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `gcsSource.inputUris` | array | Yes | GCS URIs of JSONL manifest files |
| `gcsSource.dataSchema` | string | Yes | Use `"document"` for unstructured content |
| `reconciliationMode` | enum | Yes | `"INCREMENTAL"` (add/update) or `"FULL"` (replace all) |

### JSONL Manifest Format

Each line in the JSONL file must be a valid JSON object:

```jsonl
{"id":"doc-1","jsonData":"{}","content":{"mimeType":"text/plain","uri":"gs://bucket/doc1.txt"}}
{"id":"doc-2","jsonData":"{}","content":{"mimeType":"application/pdf","uri":"gs://bucket/doc2.pdf"}}
```

#### JSONL Object Schema
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique document identifier |
| `jsonData` | string | Yes | JSON string with metadata (use `"{}"` if none) |
| `content.mimeType` | string | Yes | MIME type of document |
| `content.uri` | string | Yes | GCS URI of document |

#### Supported MIME Types
| Type | MIME Type |
|------|-----------|
| Text | `text/plain` |
| PDF | `application/pdf` |
| Word | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` |
| HTML | `text/html` |

### Response (200 OK)
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

#### Response Fields
| Field | Description |
|-------|-------------|
| `name` | Long-running operation name (save for status checking!) |
| `metadata.createTime` | When operation started |
| `metadata.updateTime` | Last update time |

### Example: Create and Upload JSONL

#### Step 1: Create JSONL locally
```bash
cat > import_documents.jsonl << 'EOF'
{"id":"mojeeb-product-doc","jsonData":"{}","content":{"mimeType":"text/plain","uri":"gs://mojeeb_test_bucket/mojeeb-product-doc.txt"}}
EOF
```

#### Step 2: Upload JSONL to GCS
```bash
gcloud storage cp import_documents.jsonl gs://mojeeb_test_bucket/import_documents.jsonl
```

#### Step 3: Import Documents
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
export PROJECT_ID="mojeeb"
export DATASTORE_ID="mojeeb-test-datastore"

curl -X POST \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents:import" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "gcsSource": {
      "inputUris": ["gs://mojeeb_test_bucket/import_documents.jsonl"],
      "dataSchema": "document"
    },
    "reconciliationMode": "INCREMENTAL"
  }'
```

### What Happens During Import?
1. **Validation**: Checks JSONL format and GCS URIs
2. **Retrieval**: Downloads documents from GCS
3. **Parsing**: Extracts text using digital parser
4. **Chunking**: Breaks documents into retrievable chunks
5. **Indexing**: Creates search index for grounding
6. **Completion**: Updates operation status

---

## API 5: Check Indexing Status

Monitors the import operation to verify indexing completion.

### Endpoint
```
GET https://discoveryengine.googleapis.com/v1/{OPERATION_NAME}
```

### URL Parameters
| Parameter | Description |
|-----------|-------------|
| `{OPERATION_NAME}` | Full operation name from import response |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
X-Goog-User-Project: {PROJECT_ID}
```

### Response - In Progress
```json
{
  "name": "projects/.../operations/import-documents-16545345710860898566",
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.ImportDocumentsMetadata",
    "createTime": "2026-01-14T15:31:38.257650Z",
    "updateTime": "2026-01-14T15:31:38.257650Z"
  }
}
```
**Status**: Processing (no `done` field or `done: false`)

### Response - Completed Successfully
```json
{
  "name": "projects/.../operations/import-documents-16545345710860898566",
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.ImportDocumentsMetadata",
    "createTime": "2026-01-14T15:31:38.257650Z",
    "updateTime": "2026-01-14T15:35:02.337055Z",
    "successCount": "1",
    "totalCount": "1"
  },
  "done": true,
  "response": {
    "@type": "type.googleapis.com/google.cloud.discoveryengine.v1.ImportDocumentsResponse",
    "errorConfig": {
      "gcsPrefix": "gs://470233234969_338402384_eu_import_document/errors16545345710860901160"
    }
  }
}
```

#### Response Fields
| Field | Description |
|-------|-------------|
| `done` | Boolean, true when complete |
| `metadata.successCount` | Number of successfully imported documents |
| `metadata.failureCount` | Number of failed documents |
| `metadata.totalCount` | Total documents processed |
| `response.errorConfig.gcsPrefix` | GCS path for error logs (if any) |

**Status**: ✓ Complete when `done: true` and `successCount > 0`

### Response - Failed
```json
{
  "name": "projects/.../operations/import-documents-XXX",
  "metadata": {
    "failureCount": "1",
    "totalCount": "1"
  },
  "done": true,
  "error": {
    "code": 400,
    "message": "Invalid document format",
    "status": "INVALID_ARGUMENT"
  }
}
```

### Example Command
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
export PROJECT_ID="mojeeb"
export OPERATION_NAME="projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/branches/0/operations/import-documents-16545345710860898566"

curl -s \
  "https://discoveryengine.googleapis.com/v1/${OPERATION_NAME}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}"
```

### Polling Best Practices

**Recommended Polling Strategy:**
- **Interval**: 2-10 seconds between polls
- **Timeout**: Maximum 2-5 minutes
- **Detection**: Check `done: true` or presence of `successCount`
- **UI Updates**: Show elapsed time and "in progress" status

**Polling Loop (Bash):**
```bash
# Poll every 10 seconds until done
while true; do
  RESPONSE=$(curl -s "https://discoveryengine.googleapis.com/v1/${OPERATION_NAME}" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" \
    -H "X-Goog-User-Project: ${PROJECT_ID}")

  if echo "$RESPONSE" | grep -q '"done": true'; then
    echo "Import completed!"
    echo "$RESPONSE" | jq '.metadata.successCount'
    break
  fi

  echo "Still processing..."
  sleep 10
done
```

**Status Messages for UI:**
- **In Progress**: "Indexing in progress... (Xs elapsed)"
- **Success**: "✓ Document indexed successfully! (X document(s))"
- **Failure**: "Import completed with X failure(s). Check logs."
- **Timeout**: "⚠ Indexing is taking longer than expected. Check back later."

---

## API 6: List Documents

Retrieves a list of all documents in the data store.

### Endpoint
```
GET https://discoveryengine.googleapis.com/v1/projects/{project}/locations/global/collections/default_collection/dataStores/{datastoreId}/branches/default_branch/documents
```

### URL Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{project}` | Project ID | `mojeeb` |
| `{datastoreId}` | Data store ID | `mojeeb-test-datastore` |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
X-Goog-User-Project: {PROJECT_ID}
```

### Query Parameters (Optional)
| Parameter | Description | Default |
|-----------|-------------|---------|
| `pageSize` | Max number of documents per page | 100 |
| `pageToken` | Token for pagination | - |

### Response (200 OK)
```json
{
  "documents": [
    {
      "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/branches/0/documents/mojeeb-product-doc",
      "id": "mojeeb-product-doc",
      "schemaId": "default_schema",
      "jsonData": "{}",
      "parentDocumentId": "mojeeb-product-doc",
      "content": {
        "mimeType": "text/plain",
        "uri": "gs://mojeeb_test_bucket/mojeeb-product-doc.txt"
      },
      "indexTime": "2026-01-14T15:33:59.231496071Z",
      "indexStatus": {
        "indexTime": "2026-01-14T15:33:59.231496071Z"
      }
    }
  ],
  "nextPageToken": "..."
}
```

#### Response Fields
| Field | Description |
|-------|-------------|
| `documents` | Array of document objects |
| `documents[].name` | Full resource name |
| `documents[].id` | Document identifier |
| `documents[].content.uri` | GCS URI of source document |
| `documents[].content.mimeType` | MIME type |
| `documents[].indexTime` | When document was indexed |
| `documents[].indexStatus` | Indexing status details |
| `nextPageToken` | Token for next page (if more results exist) |

### Example Command
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
export PROJECT_ID="mojeeb"
export DATASTORE_ID="mojeeb-test-datastore"

curl -s \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" | jq '.'
```

### With Pagination
```bash
# Get first page
curl -s \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents?pageSize=10" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}"

# Get next page using token
curl -s \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents?pageToken=NEXT_PAGE_TOKEN" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}"
```

---

## API 7: Get Single Document

Retrieves details of a specific document by ID.

### Endpoint
```
GET https://discoveryengine.googleapis.com/v1/projects/{project}/locations/global/collections/default_collection/dataStores/{datastoreId}/branches/default_branch/documents/{documentId}
```

### URL Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{project}` | Project ID | `mojeeb` |
| `{datastoreId}` | Data store ID | `mojeeb-test-datastore` |
| `{documentId}` | Document ID | `mojeeb-product-doc` |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
X-Goog-User-Project: {PROJECT_ID}
```

### Response (200 OK)
```json
{
  "name": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/branches/0/documents/mojeeb-product-doc",
  "id": "mojeeb-product-doc",
  "schemaId": "default_schema",
  "jsonData": "{}",
  "parentDocumentId": "mojeeb-product-doc",
  "content": {
    "mimeType": "text/plain",
    "uri": "gs://mojeeb_test_bucket/mojeeb-product-doc.txt"
  },
  "indexTime": "2026-01-14T15:33:55.467327Z",
  "indexStatus": {
    "indexTime": "2026-01-14T15:33:59.231496071Z"
  }
}
```

#### Response Fields
| Field | Description |
|-------|-------------|
| `name` | Full resource name |
| `id` | Document identifier |
| `schemaId` | Schema used (typically `default_schema`) |
| `jsonData` | Metadata as JSON string |
| `parentDocumentId` | Parent document ID |
| `content.mimeType` | MIME type of document |
| `content.uri` | GCS URI of source document |
| `indexTime` | Timestamp when indexed |
| `indexStatus.indexTime` | Detailed index timestamp |

### Example Command
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
export PROJECT_ID="mojeeb"
export DATASTORE_ID="mojeeb-test-datastore"
export DOCUMENT_ID="mojeeb-product-doc"

curl -s \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents/${DOCUMENT_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" | jq '.'
```

### Response - Document Not Found (404)
```json
{
  "error": {
    "code": 404,
    "message": "Document not found",
    "status": "NOT_FOUND"
  }
}
```

---

## API 8: Delete Document

Deletes a specific document from the data store.

### Endpoint
```
DELETE https://discoveryengine.googleapis.com/v1/projects/{project}/locations/global/collections/default_collection/dataStores/{datastoreId}/branches/default_branch/documents/{documentId}
```

### URL Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{project}` | Project ID | `mojeeb` |
| `{datastoreId}` | Data store ID | `mojeeb-test-datastore` |
| `{documentId}` | Document ID to delete | `doc-1768405849007` |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
X-Goog-User-Project: {PROJECT_ID}
```

### Response (200 OK)
```json
{}
```
**Empty response = successful deletion**

### Example Command
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
export PROJECT_ID="mojeeb"
export DATASTORE_ID="mojeeb-test-datastore"
export DOCUMENT_ID="doc-1768405849007"

curl -X DELETE \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents/${DOCUMENT_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}"
```

### Response - Document Not Found (404)
```json
{
  "error": {
    "code": 404,
    "message": "Document not found",
    "status": "NOT_FOUND"
  }
}
```

### Important Notes
1. **Permanent Deletion**: Document is immediately removed from the index
2. **No Undo**: Cannot recover deleted documents
3. **GCS File**: Original file in GCS is NOT deleted (only the index entry)
4. **Grounding Impact**: Document will no longer appear in grounding results immediately

### Verify Deletion
```bash
# List documents to confirm deletion
curl -s \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" | jq '.documents | length'
```

---

## API 9: Grounded Chat with Gemini

Sends queries to Gemini 2.5 Flash with grounding enabled using Vertex AI Search.

### Endpoint
```
POST https://{LOCATION}-aiplatform.googleapis.com/v1beta1/projects/{PROJECT_NUMBER}/locations/{LOCATION}/publishers/google/models/{MODEL}:generateContent
```

### URL Parameters
| Parameter | Description | Example |
|-----------|-------------|---------|
| `{LOCATION}` | Region | `us-central1` |
| `{PROJECT_NUMBER}` | Project number (not ID!) | `470233234969` |
| `{MODEL}` | Model ID | `gemini-2.5-flash` |

### Request Headers
```http
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json; charset=utf-8
X-Goog-User-Project: {PROJECT_ID}
```

### Request Body
```json
{
  "contents": [
    {
      "role": "user",
      "parts": [
        {
          "text": "What is Mojeeb and what are its key features?"
        }
      ]
    }
  ],
  "tools": [
    {
      "retrieval": {
        "vertexAiSearch": {
          "datastore": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore"
        }
      }
    }
  ],
  "generationConfig": {
    "temperature": 0.3,
    "maxOutputTokens": 1024,
    "topP": 0.95
  }
}
```

#### Request Fields
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `contents` | array | Yes | Conversation messages |
| `contents[].role` | string | Yes | `"user"` or `"model"` |
| `contents[].parts` | array | Yes | Message content parts |
| `contents[].parts[].text` | string | Yes | Actual message text |
| `tools` | array | Yes (for grounding) | Tools configuration |
| `tools[].retrieval.vertexAiSearch.datastore` | string | Yes | Full data store resource name |
| `generationConfig.temperature` | float | No | 0.0-1.0, creativity level (default: 0.9) |
| `generationConfig.maxOutputTokens` | int | No | Max response tokens (default: 8192) |
| `generationConfig.topP` | float | No | Nucleus sampling 0.0-1.0 (default: 0.95) |

### Multi-turn Conversation
```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{"text": "What is Mojeeb?"}]
    },
    {
      "role": "model",
      "parts": [{"text": "Mojeeb is an AI platform..."}]
    },
    {
      "role": "user",
      "parts": [{"text": "What are the pricing plans?"}]
    }
  ],
  "tools": [...]
}
```

### Response (200 OK)
```json
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [
          {
            "text": "Mojeeb is an advanced AI platform that offers intelligent conversational capabilities, primarily powered by Google's Gemini models. It specializes in Arabic language processing and provides enterprise-grade features for businesses.\n\nKey features of Mojeeb include:\n* Multi-Model Support\n* Grounding Capabilities\n* Arabic Language Excellence\n* Enterprise Features"
          }
        ]
      },
      "finishReason": "STOP",
      "groundingMetadata": {
        "retrievalQueries": [
          "What is Mojeeb",
          "Mojeeb key features"
        ],
        "groundingChunks": [
          {
            "retrievedContext": {
              "uri": "gs://mojeeb_test_bucket/mojeeb-product-doc.txt",
              "title": "mojeeb-product-doc",
              "text": "Mojeeb AI Platform - Product Documentation\n\nOverview:\nMojeeb is an advanced AI platform that provides intelligent conversational capabilities powered by Google's Gemini models...",
              "documentName": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore/branches/0/documents/mojeeb-product-doc"
            }
          }
        ],
        "groundingSupports": [
          {
            "segment": {
              "startIndex": 0,
              "endIndex": 132,
              "text": "Mojeeb is an advanced AI platform that offers intelligent conversational capabilities, primarily powered by Google's Gemini models."
            },
            "groundingChunkIndices": [0]
          },
          {
            "segment": {
              "startIndex": 133,
              "endIndex": 230,
              "text": "It specializes in Arabic language processing and provides enterprise-grade features for businesses"
            },
            "groundingChunkIndices": [0]
          }
        ]
      }
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 12,
    "candidatesTokenCount": 258,
    "totalTokenCount": 464,
    "trafficType": "ON_DEMAND",
    "toolUsePromptTokenCount": 83,
    "thoughtsTokenCount": 111
  },
  "modelVersion": "gemini-2.5-flash",
  "createTime": "2026-01-14T15:41:57.794569Z",
  "responseId": "Rblnacm_MNLThMIPleXMsAQ"
}
```

#### Response Fields

**Main Response**
| Field | Description |
|-------|-------------|
| `candidates[0].content.parts[0].text` | The AI's response text |
| `candidates[0].finishReason` | Why generation stopped (`STOP`, `MAX_TOKENS`, etc.) |
| `modelVersion` | Exact model version used |
| `responseId` | Unique response identifier |

**Grounding Metadata**
| Field | Description |
|-------|-------------|
| `groundingMetadata.retrievalQueries` | Queries generated to search data store |
| `groundingMetadata.groundingChunks` | Retrieved document chunks |
| `groundingMetadata.groundingChunks[].retrievedContext.uri` | Source GCS URI |
| `groundingMetadata.groundingChunks[].retrievedContext.title` | Document title |
| `groundingMetadata.groundingChunks[].retrievedContext.text` | Retrieved text chunk |
| `groundingMetadata.groundingSupports` | Maps response to sources |
| `groundingMetadata.groundingSupports[].segment` | Part of response |
| `groundingMetadata.groundingSupports[].groundingChunkIndices` | Which chunks support it |

**Usage Metadata**
| Field | Description |
|-------|-------------|
| `usageMetadata.promptTokenCount` | Input tokens |
| `usageMetadata.candidatesTokenCount` | Output tokens |
| `usageMetadata.toolUsePromptTokenCount` | Grounding overhead tokens |
| `usageMetadata.thoughtsTokenCount` | Internal reasoning tokens |
| `usageMetadata.totalTokenCount` | Total billable tokens |

### Example Command
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)

curl -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1beta1/projects/470233234969/locations/us-central1/publishers/google/models/gemini-2.5-flash:generateContent" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "X-Goog-User-Project: mojeeb" \
  -d '{
    "contents": [{
      "role": "user",
      "parts": [{
        "text": "What is Mojeeb and what are its key features?"
      }]
    }],
    "tools": [{
      "retrieval": {
        "vertexAiSearch": {
          "datastore": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore"
        }
      }
    }],
    "generationConfig": {
      "temperature": 0.3,
      "maxOutputTokens": 1024
    }
  }'
```

### Streaming Responses

Use `:streamGenerateContent` for real-time token streaming:

```bash
curl -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1beta1/projects/470233234969/locations/us-central1/publishers/google/models/gemini-2.5-flash:streamGenerateContent" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "X-Goog-User-Project: mojeeb" \
  -d '{ ... same body ... }'
```

---

## Complete Request/Response Models

### Data Store Creation

**Request Schema**
```typescript
interface CreateDataStoreRequest {
  displayName: string;                    // Required
  industryVertical: "GENERIC" | "MEDIA" | "HEALTHCARE_FHIR";  // Required
  solutionTypes: Array<"SOLUTION_TYPE_CHAT">;  // Use SOLUTION_TYPE_CHAT for grounding
  contentConfig: "CONTENT_REQUIRED" | "PUBLIC_WEBSITE" | "GOOGLE_WORKSPACE";  // Required
}
```

**Response Schema**
```typescript
interface CreateDataStoreResponse {
  name: string;           // Operation name
  done: boolean;          // Completion status
  response: {
    "@type": string;
    name: string;         // Data store resource name
    displayName: string;
    industryVertical: string;
    solutionTypes: string[];
    contentConfig: string;
    defaultSchemaId: string;
    documentProcessingConfig: {
      name: string;
      defaultParsingConfig: {
        digitalParsingConfig: {};
      };
    };
  };
}
```

### List Data Stores

**Response Schema**
```typescript
interface ListDataStoresResponse {
  dataStores: Array<{
    name: string;                       // Full resource name
    displayName: string;                // Human-readable name
    industryVertical: "GENERIC" | "MEDIA" | "RETAIL" | "HEALTHCARE";
    createTime: string;                 // ISO 8601 timestamp
    solutionTypes: Array<"SOLUTION_TYPE_CHAT" | "SOLUTION_TYPE_SEARCH">;
    contentConfig: "CONTENT_REQUIRED" | "PUBLIC_WEBSITE" | "NO_CONTENT";
    defaultSchemaId: string;
    advancedSiteSearchConfig?: {
      disableAutomaticRefresh: boolean;
    };
    documentProcessingConfig: {
      name: string;
      defaultParsingConfig: {
        digitalParsingConfig: {};
      };
      chunkingConfig?: {
        layoutBasedChunkingConfig: {
          chunkSize: number;
          includeAncestorHeadings: boolean;
        };
      };
    };
    servingConfigDataStore: {};
    billingEstimation?: {
      structuredDataSize?: string;
      unstructuredDataSize?: string;
      structuredDataUpdateTime?: string;
      unstructuredDataUpdateTime?: string;
    };
  }>;
}
```

### Import Documents

**Request Schema**
```typescript
interface ImportDocumentsRequest {
  gcsSource: {
    inputUris: string[];           // Array of gs:// URIs
    dataSchema: "document" | "custom";
  };
  reconciliationMode: "INCREMENTAL" | "FULL";
}
```

**JSONL Document Schema**
```typescript
interface DocumentImport {
  id: string;                      // Unique document ID
  jsonData: string;                // JSON string, use "{}" if no metadata
  content: {
    mimeType: string;              // MIME type
    uri: string;                   // gs:// URI
  };
}
```

**Response Schema**
```typescript
interface ImportDocumentsResponse {
  name: string;                    // Operation name
  metadata: {
    "@type": string;
    createTime: string;            // ISO 8601 timestamp
    updateTime: string;
    successCount?: string;         // Present when done
    failureCount?: string;
    totalCount?: string;
  };
  done?: boolean;                  // Present when complete
  response?: {
    "@type": string;
    errorConfig?: {
      gcsPrefix: string;           // Error logs location
    };
  };
  error?: {                        // Present if failed
    code: number;
    message: string;
    status: string;
  };
}
```

### List Documents

**Response Schema**
```typescript
interface ListDocumentsResponse {
  documents: Array<{
    name: string;                      // Full resource name
    id: string;                        // Document ID
    schemaId: string;                  // Schema ID (typically "default_schema")
    jsonData: string;                  // Metadata as JSON string
    parentDocumentId: string;          // Parent document ID
    content: {
      mimeType: string;                // MIME type
      uri: string;                     // GCS URI
    };
    indexTime: string;                 // ISO 8601 timestamp
    indexStatus: {
      indexTime: string;               // Detailed index timestamp
    };
  }>;
  nextPageToken?: string;              // Pagination token (if more results)
}
```

### Get Single Document

**Response Schema**
```typescript
interface GetDocumentResponse {
  name: string;                        // Full resource name
  id: string;                          // Document ID
  schemaId: string;                    // Schema ID
  jsonData: string;                    // Metadata as JSON string
  parentDocumentId: string;            // Parent document ID
  content: {
    mimeType: string;                  // MIME type
    uri: string;                       // GCS URI
  };
  indexTime: string;                   // ISO 8601 timestamp
  indexStatus: {
    indexTime: string;
  };
}
```

### Delete Document

**Response Schema**
```typescript
interface DeleteDocumentResponse {
  // Empty object {} on success
}
```

### Grounded Chat

**Request Schema**
```typescript
interface GenerateContentRequest {
  contents: Array<{
    role: "user" | "model";
    parts: Array<{
      text: string;
    }>;
  }>;
  tools: Array<{
    retrieval: {
      vertexAiSearch: {
        datastore: string;         // Full resource name
      };
    };
  }>;
  generationConfig?: {
    temperature?: number;          // 0.0-1.0
    maxOutputTokens?: number;
    topP?: number;                 // 0.0-1.0
    topK?: number;
  };
}
```

**Response Schema**
```typescript
interface GenerateContentResponse {
  candidates: Array<{
    content: {
      role: "model";
      parts: Array<{
        text: string;              // AI response
      }>;
    };
    finishReason: "STOP" | "MAX_TOKENS" | "SAFETY" | "RECITATION";
    groundingMetadata: {
      retrievalQueries: string[];  // Generated queries
      groundingChunks: Array<{
        retrievedContext: {
          uri: string;             // Source GCS URI
          title: string;           // Document title
          text: string;            // Retrieved chunk
          documentName: string;    // Full resource name
        };
      }>;
      groundingSupports: Array<{
        segment: {
          startIndex: number;
          endIndex: number;
          text: string;            // Response segment
        };
        groundingChunkIndices: number[];  // Source chunk indices
      }>;
    };
  }>;
  usageMetadata: {
    promptTokenCount: number;
    candidatesTokenCount: number;
    totalTokenCount: number;
    toolUsePromptTokenCount: number;
    thoughtsTokenCount: number;
  };
  modelVersion: string;
  createTime: string;
  responseId: string;
}
```

---

## Error Handling

### Common HTTP Status Codes

| Code | Status | Description |
|------|--------|-------------|
| 200 | OK | Success |
| 400 | Bad Request | Invalid request format |
| 401 | Unauthorized | Invalid or expired access token |
| 403 | Forbidden | Missing permissions or quota project |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource already exists |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |

### Error Response Format

```json
{
  "error": {
    "code": 403,
    "message": "Your application is authenticating by using local Application Default Credentials. The discoveryengine.googleapis.com API requires a quota project...",
    "status": "PERMISSION_DENIED",
    "details": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "SERVICE_DISABLED",
        "domain": "googleapis.com",
        "metadata": {
          "service": "discoveryengine.googleapis.com"
        }
      }
    ]
  }
}
```

### Common Errors and Solutions

#### 1. Token Expired (401)
```json
{
  "error": {
    "code": 401,
    "message": "Request had invalid authentication credentials."
  }
}
```
**Solution**: Get new token (expires after 1 hour)
```bash
gcloud auth print-access-token
```

#### 2. Missing Quota Project (403)
```json
{
  "error": {
    "code": 403,
    "message": "...requires a quota project...",
    "status": "PERMISSION_DENIED"
  }
}
```
**Solution**: Add `X-Goog-User-Project` header
```bash
-H "X-Goog-User-Project: mojeeb"
```

#### 3. Data Store Already Exists (409)
```json
{
  "error": {
    "code": 409,
    "message": "Data store already exists"
  }
}
```
**Solution**: Use different `dataStoreId` or delete existing one

#### 4. Model Not Found (404)
```json
{
  "error": {
    "code": 404,
    "message": "Publisher Model `...` was not found"
  }
}
```
**Solution**: Use `v1beta1` API and project number (not ID)
```
✓ Correct: /v1beta1/projects/470233234969/...
✗ Wrong:   /v1/projects/mojeeb/...
```

#### 5. Invalid JSONL Format (400)
```json
{
  "error": {
    "code": 400,
    "message": "Invalid document format"
  }
}
```
**Solution**: Ensure each line is valid JSON:
```jsonl
{"id":"doc-1","jsonData":"{}","content":{"mimeType":"text/plain","uri":"gs://bucket/doc.txt"}}
```

#### 6. CORS Error (Browser Only)
```
Access to fetch at '...' from origin '...' has been blocked by CORS policy
```
**Solution**: Use backend proxy or signed URLs for production

---

## End-to-End Example

Complete workflow from start to finish using the Mojeeb project.

### Step 1: Setup
```bash
export PROJECT_ID="mojeeb"
export PROJECT_NUMBER="470233234969"
export DATASTORE_ID="mojeeb-test-datastore"
export BUCKET="mojeeb_test_bucket"
export ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
```

### Step 2: Create Data Store
```bash
curl -X POST \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores?dataStoreId=${DATASTORE_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "displayName": "Mojeeb Test Data Store",
    "industryVertical": "GENERIC",
    "solutionTypes": ["SOLUTION_TYPE_CHAT"],
    "contentConfig": "CONTENT_REQUIRED"
  }'
```

**Result**: Save the `response.name` value

### Step 3: Upload Document to GCS
```bash
# Upload document
gcloud storage cp "sample_document.txt" "gs://${BUCKET}/mojeeb-product-doc.txt"

# Verify upload
gcloud storage ls "gs://${BUCKET}/mojeeb-product-doc.txt"
```

### Step 4: Create and Upload JSONL Manifest
```bash
# Create JSONL
cat > import_mojeeb_docs.jsonl << 'EOF'
{"id":"mojeeb-product-doc","jsonData":"{}","content":{"mimeType":"text/plain","uri":"gs://mojeeb_test_bucket/mojeeb-product-doc.txt"}}
EOF

# Upload JSONL
gcloud storage cp import_mojeeb_docs.jsonl "gs://${BUCKET}/import_mojeeb_docs.jsonl"
```

### Step 5: Import Documents
```bash
IMPORT_RESPONSE=$(curl -s -X POST \
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
  }')

# Extract operation name
OPERATION_NAME=$(echo "$IMPORT_RESPONSE" | jq -r '.name')
echo "Operation: $OPERATION_NAME"
```

### Step 6: Wait for Indexing
```bash
# Poll until complete
while true; do
  STATUS=$(curl -s \
    "https://discoveryengine.googleapis.com/v1/${OPERATION_NAME}" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" \
    -H "X-Goog-User-Project: ${PROJECT_ID}")

  if echo "$STATUS" | jq -e '.done == true' > /dev/null; then
    SUCCESS_COUNT=$(echo "$STATUS" | jq -r '.metadata.successCount')
    echo "✓ Indexing complete! Imported $SUCCESS_COUNT documents"
    break
  fi

  echo "Still indexing..."
  sleep 10
done
```

### Step 7: Test Grounded Chat
```bash
CHAT_RESPONSE=$(curl -s -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1beta1/projects/${PROJECT_NUMBER}/locations/us-central1/publishers/google/models/gemini-2.5-flash:generateContent" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "contents": [{
      "role": "user",
      "parts": [{
        "text": "What is Mojeeb and what are its key features?"
      }]
    }],
    "tools": [{
      "retrieval": {
        "vertexAiSearch": {
          "datastore": "projects/'${PROJECT_NUMBER}'/locations/global/collections/default_collection/dataStores/'${DATASTORE_ID}'"
        }
      }
    }]
  }')

# Display response
echo "$CHAT_RESPONSE" | jq '.candidates[0].content.parts[0].text'

# Display sources
echo "$CHAT_RESPONSE" | jq '.candidates[0].groundingMetadata.groundingChunks[].retrievedContext.uri'
```

### Expected Output
```
"Mojeeb is an advanced AI platform that offers intelligent conversational capabilities..."

"gs://mojeeb_test_bucket/mojeeb-product-doc.txt"
```

### Complete Script

```bash
#!/bin/bash

# Vertex AI Grounding - Complete Example
# This script demonstrates the entire workflow

set -e  # Exit on error

echo "=== Vertex AI Grounding Setup ==="

# Configuration
PROJECT_ID="mojeeb"
PROJECT_NUMBER="470233234969"
DATASTORE_ID="mojeeb-demo-$(date +%s)"
BUCKET="mojeeb_test_bucket"

# Get token
echo "Getting access token..."
ACCESS_TOKEN=$(gcloud auth print-access-token 2>/dev/null)

# Step 1: Create Data Store
echo "Step 1: Creating data store..."
curl -s -X POST \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores?dataStoreId=${DATASTORE_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "displayName": "Mojeeb Demo Data Store",
    "industryVertical": "GENERIC",
    "solutionTypes": ["SOLUTION_TYPE_CHAT"],
    "contentConfig": "CONTENT_REQUIRED"
  }' | jq '.response.name'

# Step 2: Upload document
echo "Step 2: Uploading document..."
gcloud storage cp "sample_document.txt" "gs://${BUCKET}/demo-doc.txt" 2>&1 | grep -v "WARNING"

# Step 3: Create JSONL
echo "Step 3: Creating import manifest..."
echo '{"id":"demo-doc","jsonData":"{}","content":{"mimeType":"text/plain","uri":"gs://'${BUCKET}'/demo-doc.txt"}}' > /tmp/import.jsonl
gcloud storage cp /tmp/import.jsonl "gs://${BUCKET}/import-demo.jsonl" 2>&1 | grep -v "WARNING"

# Step 4: Import
echo "Step 4: Importing documents..."
IMPORT_RESPONSE=$(curl -s -X POST \
  "https://discoveryengine.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/collections/default_collection/dataStores/${DATASTORE_ID}/branches/default_branch/documents:import" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "gcsSource": {
      "inputUris": ["gs://'${BUCKET}'/import-demo.jsonl"],
      "dataSchema": "document"
    },
    "reconciliationMode": "INCREMENTAL"
  }')
OPERATION_NAME=$(echo "$IMPORT_RESPONSE" | jq -r '.name')

# Step 5: Wait for completion
echo "Step 5: Waiting for indexing (this may take 2-3 minutes)..."
while true; do
  STATUS=$(curl -s \
    "https://discoveryengine.googleapis.com/v1/${OPERATION_NAME}" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" \
    -H "X-Goog-User-Project: ${PROJECT_ID}")

  if echo "$STATUS" | jq -e '.done == true' > /dev/null; then
    echo "✓ Indexing complete!"
    break
  fi

  sleep 10
  echo -n "."
done

# Step 6: Test chat
echo "Step 6: Testing grounded chat..."
CHAT_RESPONSE=$(curl -s -X POST \
  "https://us-central1-aiplatform.googleapis.com/v1beta1/projects/${PROJECT_NUMBER}/locations/us-central1/publishers/google/models/gemini-2.5-flash:generateContent" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json; charset=utf-8" \
  -H "X-Goog-User-Project: ${PROJECT_ID}" \
  -d '{
    "contents": [{
      "role": "user",
      "parts": [{"text": "What is Mojeeb?"}]
    }],
    "tools": [{
      "retrieval": {
        "vertexAiSearch": {
          "datastore": "projects/'${PROJECT_NUMBER}'/locations/global/collections/default_collection/dataStores/'${DATASTORE_ID}'"
        }
      }
    }]
  }')

echo "=== AI Response ==="
echo "$CHAT_RESPONSE" | jq -r '.candidates[0].content.parts[0].text'

echo ""
echo "=== Sources Used ==="
echo "$CHAT_RESPONSE" | jq -r '.candidates[0].groundingMetadata.groundingChunks[].retrievedContext.uri'

echo ""
echo "✓ Complete! Data Store ID: $DATASTORE_ID"
```

---

## References

### Official Documentation
- [Vertex AI Search API](https://cloud.google.com/generative-ai-app-builder/docs/reference/rest)
- [Discovery Engine Reference](https://cloud.google.com/generative-ai-app-builder/docs/reference/rest/v1/projects.locations.collections.dataStores)
- [Gemini 2.5 Flash](https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/2-5-flash)
- [Grounding with Vertex AI](https://cloud.google.com/vertex-ai/docs/generative-ai/grounding/ground-gemini)
- [Cloud Storage API](https://cloud.google.com/storage/docs/json_api/v1)

### Key Resources
- **Data Store Resource Name**: `projects/{number}/locations/global/collections/default_collection/dataStores/{id}`
- **Supported Regions**: us-central1, us-east1, europe-west1, asia-northeast1
- **Max Document Size**: 200 MB
- **Max Batch Import**: 100,000 documents
- **Token Expiry**: 1 hour

### Pricing (Approximate)
- **Gemini 2.5 Flash**: $0.075 per 1M input tokens, $0.30 per 1M output tokens
- **Vertex AI Search**: Storage + query costs
- **Cloud Storage**: $0.020 per GB/month (Standard class)

---

## Quick Reference

### All API Endpoints Summary

| API | Method | Endpoint |
|-----|--------|----------|
| Create Data Store | POST | `discoveryengine.googleapis.com/v1/.../dataStores` |
| List Data Stores | GET | `discoveryengine.googleapis.com/v1/.../dataStores` |
| Upload to GCS | POST | `storage.googleapis.com/upload/storage/v1/b/{bucket}/o` |
| Import Documents | POST | `discoveryengine.googleapis.com/v1/.../documents:import` |
| Check Status | GET | `discoveryengine.googleapis.com/v1/{operation}` |
| List Documents | GET | `discoveryengine.googleapis.com/v1/.../documents` |
| Get Single Document | GET | `discoveryengine.googleapis.com/v1/.../documents/{id}` |
| Delete Document | DELETE | `discoveryengine.googleapis.com/v1/.../documents/{id}` |
| Grounded Chat | POST | `aiplatform.googleapis.com/v1beta1/.../gemini-2.5-flash:generateContent` |

### Required Headers (All APIs)

```http
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json
X-Goog-User-Project: {PROJECT_ID}
```

### Critical Configuration Values

```json
{
  "industryVertical": "GENERIC",
  "solutionTypes": ["SOLUTION_TYPE_CHAT"],
  "contentConfig": "CONTENT_REQUIRED",
  "reconciliationMode": "INCREMENTAL"
}
```

---

**Document Version**: 4.1
**Last Updated**: 2026-01-14
**Tested With**: mojeeb project (470233234969)
**Status**: ✓ All 9 APIs tested and working

**Changelog**:
- **v4.1**: Enhanced API 5 (Check Indexing Status) with polling best practices, recommended intervals, and UI status messages
- **v4.0**: Added List Data Stores API with complete TypeScript interfaces for dropdown data store selection
- **v3.1**: Updated Create Data Store API to use `SOLUTION_TYPE_CHAT` and clarified `CONTENT_REQUIRED` for document-based grounding
- **v3.0**: Added List Documents, Get Single Document, Delete Document APIs
- **v2.0**: Complete API reference with all endpoints
- **v1.0**: Initial documentation
