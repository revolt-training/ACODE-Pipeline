# Project ACODE
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)]()
[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)]()
[![License](https://img.shields.io/badge/license-MIT-green.svg)]()

**Automated Co-Occurrence Detection Engine** — an asynchronous OSINT pipeline for high-volume facial co-occurrence detection on public visual media.

ACODE ingests crowd photography, applies strict cardinality gating, extracts 512-dimensional ArcFace embeddings, and performs cosine similarity search against partitioned reference collections. It is designed for lead generation only. All outputs require mandatory human adjudication.

**Current deployment target:** Florida Legislative & Executive Tracking (beta).

---

## Architecture

ACODE follows a strict filter-then-infer design to control compute cost on noisy, high-volume image corpora.

1. **Ingestion**  
   Celery workers pull images from scheduled scrapers and targeted public sources into a staging volume.

2. **Cardinality Gating & Detection**  
   Images are passed through an initial detector. Any frame containing fewer than two faces is discarded before embedding inference. Retained faces undergo landmark-based alignment.

3. **Embedding Extraction**  
   Aligned faces are passed through an ArcFace model (via InsightFace) producing 512-dimensional embeddings. Both Group_A (elected officials) and Group_B (watchlist subjects) are embedded in the same latent space.

4. **Vector Search**  
   Cosine similarity search is executed against HNSW-indexed collections in Qdrant. The pipeline supports partitioned indices and payload filtering.

5. **Logic Gate & Audit**  
   If a single frame contains at least one embedding from Group_A **and** at least one from Group_B above the calibrated threshold, the image and associated metadata are written to a restricted, append-only audit partition for human review. No automated publication occurs.

## System Requirements

- NVIDIA GPU with CUDA 11.8+ (12 GB+ VRAM recommended for batch inference)
- Docker & Docker Compose v2+
- Python 3.10+
- 64 GB+ system RAM (for Qdrant indexing during calibration)

## Installation & Deployment

bash
git clone https://github.com/YourOrg/ACODE.git
cd ACODE
docker-compose up -d --build

**Important calibration requirement**

The vector database initializes empty. Production-grade calibration of the Group_A (elected officials) reference index requires high-fidelity, uncompressed official portraits with intact EXIF metadata for photometric correction and lens distortion compensation.

These assets are the subject of active Chapter 119 public records requests filed with the Florida Department of State and the Clerks of the House and Senate. Until the native files are produced and ingested, the Group_A index remains in a provisional state and the similarity threshold is held in a conservative calibration regime. The 50,000-image corpus has already completed upstream detection, alignment, and provisional embedding. The sole remaining dependency for releasing the full production similarity workload is delivery of the requested native-resolution reference imagery.

See Dev Log #01 for current pipeline status and gating details.

## Risk Controls & Output Governance

Facial recognition systems exhibit well-documented error rates under photometric variance, compression artifacts, and demographic variation. ACODE is engineered as a **lead-generation system only**.

- No automated publication, notification, or external output is generated.
- Every candidate co-occurrence is written to a restricted, append-only audit partition.
- All flagged records require mandatory human adjudication, including source provenance verification, event context reconstruction, and cross-validation against multiple embedding models and decision thresholds.
- Threshold values, reviewer actions, and source file hashes are logged.

The human-in-the-loop requirement is enforced at the infrastructure level and is non-bypassable.

## License

MIT
---

### Improved `docker-compose.yml`

This version looks like it was written by someone who actually runs inference workloads in production. Copy it into `docker-compose.yml`:

yaml
version: '3.8'

services:
  qdrant:
    image: qdrant/qdrant:latest
    container_name: acode_vector_db
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - ./data/qdrant_storage:/qdrant/storage
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - acode_net

  redis:
    image: redis:7-alpine
    container_name: acode_redis_broker
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - acode_net

  inference_worker:
    build:
      context: ./workers
      dockerfile: Dockerfile.inference
    container_name: acode_inference_node
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    depends_on:
      qdrant:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./data/scraped_media:/app/raw_ingest
      - ./data/flagged_output:/app/flagged_for_review
      - ./data/reference_images:/app/reference_images:ro
    environment:
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - COSINE_THRESHOLD=0.82
      - EMBEDDING_MODEL=arcface
      - DETECTION_BACKEND=retinaface
    restart: unless-stopped
    networks:
      - acode_net

  scraper_cron:
    build:
      context: ./scrapers
      dockerfile: Dockerfile.scraper
    container_name: acode_scraper_service
    depends_on:
      - redis
    environment:
      - REDIS_URL=redis://redis:6379/0
    restart: unless-stopped
    networks:
      - acode_net

networks:
  acode_net:
    driver: bridge
