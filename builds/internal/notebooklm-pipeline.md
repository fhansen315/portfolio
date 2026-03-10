# NotebookLM Pipeline

**AI Content Processing Automation** | Automation | 2026

---

## Overview

Built automated pipeline for extracting and processing YouTube transcripts, enabling efficient content ingestion for AI knowledge bases.

---

## Problem

Manual content processing was time-consuming:
- YouTube videos contain valuable information
- Manual transcript extraction is tedious
- No structured format for AI ingestion
- Playlist processing required individual handling

---

## Solution

Created end-to-end content processing pipeline:

### Transcript Extraction
- YouTube API integration
- Playlist batch processing
- VTT subtitle parsing
- Multiple extraction methods (standard, innertube, v2)

### Format Conversion
- VTT to CSV conversion
- Structured data output
- Timestamp preservation
- Speaker identification (where available)

### Pipeline Orchestration
- Playlist YAML configuration
- Batch processing support
- Output organization
- Error handling and retry logic

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Extraction Methods | 3 |
| Output Format | CSV |
| Batch Processing | Supported |
| Pipeline Status | Working |

---

## Technologies

`Python` `YouTube API` `VTT Parsing` `CSV` `YAML Configuration`

---

## Key Deliverables

- extract_transcripts.py - Standard extraction
- extract_transcripts_v2.py - Enhanced extraction
- extract_transcripts_innertube.py - Alternative method
- vtt_to_csv.py - Format converter
- pipeline.py - Orchestration script
- playlists.yaml - Configuration file
- output/ - Processed transcripts

---

## Impact Statement

> Automated YouTube content extraction, enabling efficient knowledge ingestion for AI systems from video playlists.

---

**Source**: Automation/NotebookLM/ | **Status**: Complete
