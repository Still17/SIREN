# Architecture Overview

SIREN is built as a modular, cloud-agnostic pipeline optimized for high-frequency data ingestion and real-time analysis.

## Core Components

- **Data Ingestion**
  Continuous ingestion of social data streams with engagement-based filtering and deduplication.

- **Processing & Filtering**
  Real-time preprocessing, normalization, and signal validation.

- **LLM Classification Layer**
  AI-driven classification for sentiment polarity, analytical depth, and topic relevance.

- **Credibility Scoring**
  Behavioral and engagement-based author evaluation to weight influence and reduce noise.

- **Aggregation & Analytics**
  Time-windowed aggregation of weighted sentiment signals across multiple horizons.

## Infrastructure Characteristics

- Horizontally scalable compute
- Stateless processing workers
- Managed databases and analytics services
- Parallel LLM inference workloads
