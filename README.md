# a1100

## Internal AI-Assisted Consultation Tool (Nose-Surgery Similar Cases)

### 1) Project Goal
Build an internal AI-assisted consultation tool for the clinic:

- Staff takes a photo of a new customer (BEFORE)
- System finds past customers with similar facial structure (from your internal catalog)
- System returns AFTER images + case metadata of the most similar patients to support consultation and expectation-setting

**Primary Output**

- Top-K similar past cases (e.g., K=5)
- Similarity score (0–1)
- AFTER image(s) and metadata (procedure type, nose style, date, notes if allowed)

**Non-goals (MVP)**

- Not generating “simulated after” images (GAN/Diffusion) in MVP (risk/legal/expectation issues)
- Not automatic medical recommendation / guarantee outcomes
- Not public-facing search or marketing tool

### 2) Scope, Users, and Success Metrics
**Users**

- Consultants / reception: upload photo, view similar cases
- Admin: manage catalog, remove cases, re-index, audit logs

**Core Functional Requirements (MVP)**

- Upload new photo (BEFORE)
- Search similar faces from historical catalog
- Filter by: gender, procedure type (nose), optional nose style
- Display results with similarity scores + after images
- Error handling: no face, multiple faces, low quality

**Success Metrics (practical, measurable)**

**Quality**

- Precision@5 (human-rated): ≥ 80% “usable matches”
- Rejection rate for bad images: correct rejection ≥ 90%

**Performance**

- Search latency: < 2s for top-5 on clinic network
- Ingestion time per case: < 2s (CPU acceptable)

**Operational**

- 99% successful requests (excluding invalid images)
- Full audit log of access and searches

### 3) Data Requirements & Governance
**Data Needed per Case (catalog record)**

- BEFORE photo (required)
- AFTER photo(s) (required)
- Metadata:
  - procedure_type (e.g., nose augmentation)
  - nose_style (clinic-defined taxonomy)
  - gender, age_range (optional)
  - date, surgeon/team (internal only)
  - consent status (required)

**Photo Standardization (strongly recommended)**

- Front-facing, neutral expression, no filter
- Good lighting, no glasses/mask if possible
- One face per image

**Privacy / Compliance (internal best-practice)**

- Require written consent for internal reuse in consultation
- Role-based access (consultant vs admin)
- Store images encrypted at rest
- Log every search + who accessed which cases
- Optional: watermark/blur for additional privacy

### 4) Recommended Technical Approach
This is a classic Face Similarity Search system:

- Face detection + alignment
- Face embedding (convert face to numeric vector)
- Store vectors in a Vector Database
- Query by similarity (cosine distance) + metadata filters
- Return matched cases + after photos

**Why this approach works**

- Robust to lighting/background differences
- Very fast retrieval even with tens/hundreds of thousands of cases
- Doesn’t require training from scratch

### 5) Model & Component Choices
**Face Detection + Embedding Model (Recommended)**

- InsightFace (ArcFace embeddings)
- Widely used for face embeddings
- Good accuracy and speed
- Easy integration in Python
- Suggested model pack: buffalo_l
- Provides detection + landmarks + 512-dim embedding

**Vector Database (Recommended)**

- Qdrant
- Simple operations, excellent performance
- Supports payload filters (gender/procedure)
- Easy Docker deployment for internal systems

(Alternatives: Milvus for very large scale; FAISS for prototype only)

### 6) System Architecture (MVP)
**High-level flow**

UI (web/tablet) → API service (FastAPI)

Storage: images in object storage (MinIO/S3) or secure file server

Metadata: PostgreSQL

Vector index: Qdrant

**Runtime steps**

- Upload image
- Detect face → align → embedding vector
- Query Qdrant for top-K similar vectors with filters
- Fetch metadata + after image URLs
- Render results in UI

### 7) Implementation Plan (Step-by-step)
**Phase 0 — Foundation & Standards**

**Deliverables**

- Photo capture guideline (1-page SOP)
- Consent template finalized
- Metadata taxonomy (nose styles, procedures)

**Tech**

- None (process)

**Phase 1 — Data & Storage Layer**

**Deliverables**

- Image storage (MinIO or secure NAS path)
- PostgreSQL schema for cases
- Admin upload tool for historical cases

**Tech**

- PostgreSQL
- MinIO (S3-compatible) or file server

**Phase 2 — AI Service (Embedding Pipeline)**

**Deliverables**

- Face detection + alignment module
- Embedding generation module
- Quality gates:
  - reject “no face”
  - reject “multiple faces”
  - reject “too blurry / too small face” (basic heuristics)

**Tech**

- Python, OpenCV/Pillow
- InsightFace (ArcFace)

**Phase 3 — Vector DB Integration**

**Deliverables**

- Qdrant collection created (cosine distance)
- Upsert vectors with payload metadata
- Re-index command (rebuild from DB)

**Tech**

- Qdrant (Docker)

**Phase 4 — Search API + Business Filters**

**Deliverables**

- /search endpoint: returns top-K matches
- Filters:
  - gender match (default on)
  - procedure_type = nose (required)
  - optional nose_style

**Notes**

- Use similarity threshold (e.g., 0.85) to avoid bad matches
- Return “no good match” message if threshold not met

**Phase 5 — UI for Consultants**

**Deliverables**

- Upload/capture screen
- Results screen:
  - New photo on left
  - Top 3–5 after results on right
  - Similarity score + summary tags

- Disclaimer banner (“reference only…”)

**Tech**

- React / Next.js (recommended) OR simple internal web UI

**Phase 6 — QA, Evaluation, and Tuning**

**Deliverables**

- Human evaluation set (50–200 queries)
- Tune:
  - threshold
  - top-K
  - filters defaults

- Logging dashboard for usage + failure cases

### 8) Technical Notes (Deep Engineering)
**8.1 Embedding and Similarity**

- Generate 512-d float vector per face
- Normalize vector (L2) before insert/search
- Similarity: cosine

**8.2 Quality Gates (important)**
Reject or warn if:

- Face bbox too small (e.g., < 120px width)
- Blurry (variance of Laplacian below threshold)
- Extreme pose (yaw/pitch beyond reasonable range if available)

**8.3 Metadata Filtering Strategy**
Do not rely only on “face similarity”; apply payload filters:

- procedure_type = nose
- gender = same
- optionally face_shape_category if you add later

This increases trust and reduces “weird matches”.

**8.4 Evaluation Method (internal)**

- Create a test set of “new customer photos”
- Staff rates returned top-5:
  - Good / Acceptable / Bad

- Track Precision@5 and “first good match rank”

**8.5 Failure Modes and Handling**

- No face detected → instruct to retake photo
- Multiple faces → “only 1 person in frame”
- Low similarity → show “no close match found” + allow manual browsing by nose style

### 9) Proposed Stack (Dev → Deploy → Operate)
**Development Stack**

**Backend**

- Python 3.10+
- FastAPI + Uvicorn
- InsightFace + OpenCV
- Pydantic settings + structured logging

**Databases**

- PostgreSQL (metadata)
- Qdrant (vector search)

**Storage**

- MinIO (recommended) or on-prem file server with signed URLs

**Frontend**

- Next.js (React) internal web portal

**Auth**

- Internal SSO or simple JWT/LDAP integration

**Deployment (Internal / On-prem)**

**MVP Deployment**

- Docker Compose on an internal server:
  - api-service
  - qdrant
  - postgres
  - minio

**Scale-up Option**

- Kubernetes (if multiple clinics / heavy concurrency)
- Separate node pool with optional GPU

**Hardware**

- CPU is enough for MVP (embedding is fast)
- GPU optional if you later add heavier analysis/simulation

**Operations & Runbook**

**Monitoring**

- Metrics:
  - request latency
  - face-detect fail rate
  - low-similarity rate
  - Qdrant query time

**Tools**

- Prometheus + Grafana (ideal)
- Or simple logs + ELK / Loki

**Logging & Audit**

- Log:
  - who searched (user id)
  - when
  - returned case ids (or hashed ids)

- Separate audit logs from app logs

**Backups**

- PostgreSQL daily backup
- Qdrant snapshot schedule
- MinIO bucket replication or periodic backup

**Security**

- TLS internal
- RBAC roles: consultant/admin
- Encrypt at rest (disk or storage-level encryption)
- Strict network rules (only internal LAN)

**Maintenance**

- “Re-index pipeline” command:
  - rebuild vectors if model updated

- “Case removal”:
  - remove from Postgres + Qdrant + storage (for consent withdrawal)

### 10) Suggested API Contracts (Concrete)
**POST /cases (admin ingestion)**

Input: before/after images + metadata

Output: case_id, indexing_status

**POST /search**

Input: new before image + filters

Output:

```json
{
  "query_id": "uuid",
  "results": [
    {
      "case_id": "case_123",
      "similarity": 0.92,
      "after_image_url": "https://...",
      "procedure_type": "nose",
      "nose_style": "high-bridge"
    }
  ],
  "message": "ok"
}
```

### 11) Optional Enhancements (After MVP)

- Face attribute tagging (face shape categories) to improve filtering
- Nose-region feature extraction (landmark ratios) to better match “nose compatibility”
- Simulation module (GAN/Diffusion) only with strict disclaimers + legal review
- Multi-angle support (front + 45°) for higher accuracy

### Summary Recommendation
For an internal clinic system, the best balance of accuracy / speed / safety is:

- InsightFace (ArcFace embeddings) for face similarity
- Qdrant for vector search with metadata filters
- FastAPI + Postgres + MinIO for reliable internal deployment
- Docker Compose MVP → Kubernetes only if you scale to multiple branches

If you want, I can also provide:

- A full Jira task breakdown aligned to the phases above (backend/frontend/devops)
- A “golden path” repo structure with config templates (docker-compose, env, migration scripts)
- A minimal reference implementation skeleton (FastAPI + InsightFace + Qdrant)
