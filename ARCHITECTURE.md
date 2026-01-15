# SIREN — System Architecture

This document describes the **high-level architecture** of the SIREN sentiment intelligence system.

The system is designed for **cloud-native experimentation**, scalability, and infrastructure benchmarking, with a focus on real-time data processing and AI-driven analysis.

---

## Architectural Overview

SIREN follows a modular, pipeline-based architecture:

**Ingestion → Preprocessing → LLM Analysis → Scoring → Aggregation → Research Output**

Each stage is independently scalable and loosely coupled, allowing flexible experimentation with infrastructure and model configurations.

---

## Core Components

### 1. Ingestion Layer
- Collects high-frequency crypto-related social data
- Performs deduplication and rate normalization
- Designed for burst traffic and variable load

### 2. Preprocessing Layer
- Cleans and normalizes raw text data
- Extracts metadata and engagement signals
- Prepares structured inputs for AI analysis

### 3. LLM Analysis Layer
- Applies large language models for:
  - sentiment polarity detection
  - analytical depth classification
  - topic relevance assessment
- Designed to support multiple LLM providers

### 4. Scoring Layer
- Combines AI outputs with behavioral and engagement metrics
- Applies credibility-weighted scoring logic
- Reduces noise from low-quality or manipulative sources

### 5. Aggregation Layer
- Aggregates scores across multiple rolling time windows
- Produces sentiment flow metrics and pressure indicators
- Enables multi-timeframe analysis

### 6. Research Output Layer
- Outputs structured sentiment intelligence signals
- Designed for downstream research, monitoring, and experimentation
- No direct user-facing UI required at this stage

---

## Deployment and Scalability

- Stateless processing components where possible
- Horizontal scaling for ingestion and AI inference workloads
- Cloud-managed storage and analytics services for efficiency
- Designed for rapid iteration across different cloud environments

---

## Potential Bottlenecks

- High-throughput data ingestion under burst conditions
- Cost and latency of LLM inference at scale
- Aggregation performance across large rolling windows

These constraints are actively evaluated through cloud-based benchmarking.

---

## Security Considerations

- No hardcoded credentials or secrets
- Separation of data ingestion, processing, and inference layers
- Designed to integrate with cloud-native IAM and secret management systems

---

## Infrastructure Rationale

SIREN requires cloud infrastructure to:
- scale ingestion and processing dynamically
- efficiently run AI inference workloads
- store and analyze large volumes of structured and unstructured data
- benchmark multiple infrastructure and LLM configurations

This architecture supports infrastructure experimentation as part of cloud startup and research grant programs.
