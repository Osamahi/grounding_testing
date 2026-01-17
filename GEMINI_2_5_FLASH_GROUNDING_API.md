# Gemini 2.5 Flash with Vertex AI Search Grounding - REST API

## Successfully Tested on 2026-01-14

This document records the working REST API endpoint and request format for using Gemini 2.5 Flash with Vertex AI Search grounding.

---

## REST API Endpoint

```
POST https://us-central1-aiplatform.googleapis.com/v1beta1/projects/{PROJECT_NUMBER}/locations/us-central1/publishers/google/models/gemini-2.5-flash:generateContent
```

### Endpoint Components

- **Base URL**: `https://us-central1-aiplatform.googleapis.com`
- **API Version**: `v1beta1` (Required for grounding features)
- **Location**: `us-central1` (or other supported regions)
- **Model**: `gemini-2.5-flash`
- **Method**: `generateContent` (for unary/complete responses)

### Alternative Method
- Use `:streamGenerateContent` for streaming responses

---

## Required Headers

```http
Authorization: Bearer {ACCESS_TOKEN}
Content-Type: application/json; charset=utf-8
X-Goog-User-Project: {PROJECT_ID}
```

### Getting Access Token
```bash
gcloud auth print-access-token
```

---

## Request Body Format

```json
{
  "contents": [{
    "role": "user",
    "parts": [{
      "text": "Your question here"
    }]
  }],
  "tools": [{
    "retrieval": {
      "vertexAiSearch": {
        "datastore": "projects/{PROJECT_NUMBER}/locations/global/collections/default_collection/dataStores/{DATASTORE_ID}"
      }
    }
  }]
}
```

### Key Parameters

- **contents**: Array of conversation turns
  - **role**: "user" or "model"
  - **parts**: Array of content parts
    - **text**: The actual query/prompt

- **tools**: Array of tools (for grounding)
  - **retrieval**: Retrieval configuration
    - **vertexAiSearch**: Vertex AI Search configuration
      - **datastore**: Full resource name of your data store

---

## Complete Working Example

### Environment Variables
```bash
export ACCESS_TOKEN=$(~/google-cloud-sdk/bin/gcloud auth print-access-token 2>/dev/null)
export PROJECT_NUMBER="470233234969"
export PROJECT_ID="mojeeb"
export DATASTORE_ID="mojeeb-test-datastore"
```

### cURL Command
```bash
curl -X POST \
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
          "datastore": "projects/470233234969/locations/global/collections/default_collection/dataStores/mojeeb-test-datastore"
        }
      }
    }]
  }'
```

---

## Response Structure

### Successful Response

```json
{
  "candidates": [{
    "content": {
      "role": "model",
      "parts": [{
        "text": "Mojeeb is an advanced AI platform that offers intelligent conversational capabilities..."
      }]
    },
    "finishReason": "STOP",
    "groundingMetadata": {
      "retrievalQueries": [
        "What is Mojeeb",
        "Mojeeb key features"
      ],
      "groundingChunks": [{
        "retrievedContext": {
          "uri": "gs://mojeeb_test_bucket/mojeeb-product-doc.txt",
          "title": "mojeeb-product-doc",
          "text": "Mojeeb AI Platform - Product Documentation...",
          "documentName": "projects/470233234969/.../documents/mojeeb-product-doc"
        }
      }],
      "groundingSupports": [{
        "segment": {
          "startIndex": 132,
          "endIndex": 230,
          "text": "It specializes in Arabic language processing..."
        },
        "groundingChunkIndices": [0]
      }]
    }
  }],
  "usageMetadata": {
    "promptTokenCount": 12,
    "candidatesTokenCount": 258,
    "totalTokenCount": 464
  },
  "modelVersion": "gemini-2.5-flash"
}
```

### Response Breakdown

#### 1. candidates[].content
The actual generated response from the model.

#### 2. groundingMetadata
Contains information about how the response was grounded:

- **retrievalQueries**: Queries generated to search the datastore
- **groundingChunks**: Retrieved document chunks
  - **retrievedContext**: Full chunk details
    - **uri**: Original document location in GCS
    - **title**: Document identifier
    - **text**: Extracted text from the document
    - **documentName**: Full resource name

- **groundingSupports**: Maps response segments to source chunks
  - **segment**: Part of the response text
    - **startIndex**: Start position in response
    - **endIndex**: End position in response
    - **text**: The actual text segment
  - **groundingChunkIndices**: Which chunks support this segment

#### 3. usageMetadata
Token usage statistics for billing and monitoring.

---

## How Grounding Works (Step-by-Step)

1. **User sends query** → "What is Mojeeb and what are its key features?"

2. **Model generates retrieval queries** →
   - "What is Mojeeb"
   - "Mojeeb key features"

3. **Vertex AI Search retrieves relevant chunks** →
   - Searches the data store
   - Returns most relevant document chunks
   - Includes document URI, title, and text

4. **Model generates grounded response** →
   - Uses retrieved chunks as context
   - Generates accurate, factual response
   - Maps response segments to source chunks

5. **Response includes grounding metadata** →
   - Shows which documents were used
   - Indicates which parts are grounded
   - Provides source attribution

---

## Test Results

### Test 1: Irrelevant Query
**Query**: "What are Dr Bahy social media links?"

**Result**:
- Model generated retrieval queries
- No relevant chunks found (document doesn't contain this info)
- Empty response returned
- ✓ System correctly handled missing information

### Test 2: Relevant Query
**Query**: "What is Mojeeb and what are its key features?"

**Result**:
- Retrieved document: `mojeeb-product-doc.txt`
- Generated comprehensive answer with:
  - Multi-Model Support details
  - Grounding Capabilities
  - Arabic Language Excellence
  - Enterprise Features
  - Technical Specifications
- 6 grounding supports showing source attribution
- ✓ Perfect grounding implementation

---

## Important Notes

### 1. API Version
**Must use `v1beta1`** - The grounding feature requires the beta API version.

```
✓ Correct: /v1beta1/projects/...
✗ Wrong:   /v1/projects/...
```

### 2. Project Identifier
Use **PROJECT_NUMBER** (not project ID) in the endpoint path:

```
✓ Correct: projects/470233234969/locations/...
✗ Wrong:   projects/mojeeb/locations/...
```

But use **PROJECT_ID** in the `X-Goog-User-Project` header:

```
✓ Correct: X-Goog-User-Project: mojeeb
```

### 3. Data Store Resource Name
Must be the full resource name from the data store creation:

```
projects/{PROJECT_NUMBER}/locations/global/collections/default_collection/dataStores/{DATASTORE_ID}
```

### 4. Supported Regions
- us-central1
- us-east1
- us-west1
- europe-west1
- asia-northeast1
- And more (see Vertex AI docs)

### 5. Model Availability
Gemini 2.5 Flash is available in `v1beta1` API. Check the latest docs for GA availability.

---

## Common Errors and Solutions

### Error: "Publisher Model was not found"
**Cause**: Wrong API version or model name

**Solution**:
- Use `v1beta1` instead of `v1`
- Use exact model name: `gemini-2.5-flash`

### Error: "Permission Denied" or quota issues
**Cause**: Missing X-Goog-User-Project header

**Solution**: Add header with project ID
```
-H "X-Goog-User-Project: mojeeb"
```

### Error: Empty response with grounding
**Cause**: No relevant content in data store

**Solution**:
- Verify documents were imported successfully
- Check import operation status
- Ensure query is relevant to document content

---

## Next Steps - Production Usage

### 1. Add Error Handling
```bash
response=$(curl -s -w "\n%{http_code}" ...)
http_code=$(echo "$response" | tail -n1)
body=$(echo "$response" | head -n-1)

if [ "$http_code" -eq 200 ]; then
  echo "Success: $body"
else
  echo "Error $http_code: $body"
fi
```

### 2. Implement Streaming
For real-time responses, use `streamGenerateContent`:
```bash
curl -X POST ".../gemini-2.5-flash:streamGenerateContent" ...
```

### 3. Add Generation Configuration
```json
{
  "contents": [...],
  "tools": [...],
  "generationConfig": {
    "temperature": 0.7,
    "maxOutputTokens": 1024,
    "topP": 0.95
  }
}
```

### 4. Monitor Token Usage
Track `usageMetadata` for cost optimization:
- `promptTokenCount`: Input tokens
- `candidatesTokenCount`: Output tokens
- `toolUsePromptTokenCount`: Grounding overhead

---

## Resources

- **Vertex AI Documentation**: https://cloud.google.com/vertex-ai/docs
- **Gemini 2.5 Flash**: https://cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/2-5-flash
- **Grounding Guide**: https://cloud.google.com/vertex-ai/generative-ai/docs/grounding/grounding-with-vertex-ai-search
- **Data Store Creation**: See `DOCUMENT_UPLOAD_PROCESS.md`

---

## Summary

✓ **Endpoint**: `v1beta1` API with `gemini-2.5-flash` model
✓ **Authentication**: Bearer token + X-Goog-User-Project header
✓ **Grounding**: Works via `tools.retrieval.vertexAiSearch`
✓ **Response**: Includes grounded answer + grounding metadata
✓ **Attribution**: Full source tracking via groundingSupports

This implementation provides true RAG (Retrieval-Augmented Generation) with:
- Automatic chunking and indexing
- Semantic search and retrieval
- Source attribution
- Factual, grounded responses
