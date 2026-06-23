# Architecture Overview — ACODE v0.9-beta

ACODE is an asynchronous, containerized OSINT pipeline designed for high-volume facial co-occurrence detection on public visual media. It follows a strict **filter-then-infer** design to control compute cost while maintaining a non-bypassable human adjudication layer.

The system is currently deployed in beta against Florida legislative and executive branch imagery. Production similarity search remains gated pending delivery of native-resolution official portraits requested under Chapter 119.

## High-Level Design Principles

- **Early filtering**: Discard the majority of images before expensive embedding inference.
- **Deterministic audit trail**: Every candidate co-occurrence is written to an append-only partition with source hashing.
- **Human judgment as infrastructure**: Automated detection never becomes automated assertion.
- **Calibration transparency**: Similarity thresholds and embedding versions are explicit, versioned, and logged.
- **Minimal external dependencies** at inference time.

## Core Components

| Component          | Technology                  | Responsibility                              | Scaling Notes                     |
|--------------------|-----------------------------|---------------------------------------------|-----------------------------------|
| Message Broker     | Redis 7                     | Task queue for ingestion and inference      | Horizontal via Redis Cluster     |
| Task Workers       | Celery + Python 3.10+       | Orchestration of detection and embedding    | Horizontal worker fleet          |
| Inference Engine   | InsightFace (ArcFace)       | Face detection, alignment, 512-d embedding  | GPU-bound; one container per GPU |
| Vector Database    | Qdrant                      | HNSW-indexed cosine similarity search       | Persistent volume; payload filtering supported |
| Scraper Scheduler  | Custom Celery beat service  | Scheduled ingestion from public sources     | Low frequency, low resource      |
| Audit Store        | Append-only filesystem      | Write-once storage for flagged records      | Immutable by design              |

## Pipeline Data Flow

Raw Image
   │
   ▼
[Ingestion Worker]
   │
   ▼
[MTCNN Cardinality Gate]
   │   (discard if < 2 faces)
   ▼
[RetinaFace Landmarking + Alignment]
   │
   ▼
[ArcFace 512-d Embedding Extraction]
   │
   ├──────────────────────┐
   │                      │
   ▼                      ▼
[Group_A Index]      [Group_B Index]
(Qdrant)             (Qdrant)
   │                      │
   └──────────┬───────────┘
              ▼
   [Cosine Similarity + Threshold Check]
              │
              ▼
   [MATCH(Group_A) ∧ MATCH(Group_B)]
              │
              ▼
   [Append-only Audit Partition]
              │
              ▼
   [Human Adjudication Queue]

### Stage Details

**1. Ingestion**  
Celery workers pull images from scheduled scrapers into a shared staging volume (`/app/raw_ingest`). Metadata (source URL, timestamp, event context when available) is attached as task payload.

**2. Cardinality Gating (MTCNN)**  
A lightweight MTCNN detector runs first on CPU-bound workers. Any image returning fewer than two face detections is dropped. This stage eliminates ~68% of typical rally/event photography before any GPU work occurs.

**3. Landmarking & Alignment (RetinaFace)**  
Surviving multi-face images are passed to a RetinaFace model for high-precision five-point landmark detection. A similarity transform is applied to produce eye-level aligned crops.

**4. Embedding Extraction (ArcFace)**  
Aligned faces are embedded using an ArcFace model (ResNet-100 backbone via InsightFace) producing 512-dimensional vectors. Embeddings are generated for both the incoming image faces and (during calibration) the reference sets.

**5. Vector Search (Qdrant)**  
Cosine similarity search is performed against two partitioned collections:
- `Group_A`: Elected officials (currently provisional; pending native-resolution calibration)
- `Group_B`: Watchlist subjects

Qdrant’s HNSW index and payload filtering allow efficient queries with optional metadata constraints (date ranges, event types, etc.).

**6. Logic Gate & Audit**  
If a single frame contains at least one embedding from Group_A **and** at least one from Group_B above the active cosine threshold, the original image, aligned crops, embeddings, similarity scores, and metadata are written to a restricted append-only directory (`/app/flagged_for_review`). Cryptographic hashes of source files are recorded.

**7. Human Adjudication**  
Flagged records enter a manual review workflow. Reviewers must:
- Verify image provenance and authenticity
- Reconstruct event context from public records
- Cross-check against multiple embedding models and threshold variants
- Record final disposition with justification

No automated publication or external notification is possible.

## Vector Index & Calibration Strategy

The system maintains separate collections for Group_A and Group_B to allow independent threshold tuning and re-calibration.

- **Provisional mode** (current beta): Group_A embeddings are derived from publicly available compressed JPEG portraits. This introduces measurable embedding drift.
- **Production mode** (gated): Requires native, uncompressed digital files with intact EXIF metadata for photometric correction and lens distortion compensation.

As of June 2026, production mode is blocked pending Chapter 119 production of official legislative and executive portraits from the Florida Department of State and the Clerks of the House and Senate. The 50,000-image corpus has already completed stages 1–4 and resides in staging storage. Release of the full similarity search workload is contingent solely on delivery and re-embedding of the requested native reference assets.

## Security & Audit Controls

- All flagged records are written to an append-only filesystem partition with file-level hashing.
- The human adjudication step is enforced at the infrastructure level (no code path exists to bypass it).
- Reference image volumes are mounted read-only inside the inference container.
- Environment variables controlling thresholds and model versions are logged at worker startup.
- Container images use pinned base images and are built through CI.

## Deployment Topology

The system is orchestrated via Docker Compose for the beta deployment:

- `qdrant` — persistent vector store
- `redis` — task broker
- `inference_worker` — GPU-enabled inference (ArcFace + detection models)
- `scraper_cron` — scheduled ingestion

Horizontal scaling of inference workers is supported by adding additional GPU-equipped containers. The current beta runs with a single inference node.

## Current Limitations & Known Constraints

- Group_A calibration is in provisional state due to pending Chapter 119 production of native official portraits.
- Similarity thresholds are currently set conservatively to compensate for embedding drift in the provisional reference set.
- Group_B watchlist composition and update cadence are managed outside the public repository for operational security reasons.
- The system does not perform real-time or streaming inference; all workloads are batch-oriented.

These constraints are documented here so that contributors and reviewers understand the exact boundary between what the pipeline can currently do and what it will be able to do once the requested public records are produced.
