# DataScrub AI: System Architecture & Visual Pipeline

This document outlines the architectural logic and decision-making pipeline used by **DataScrub AI** to ingest raw business requests, profile data structures, enforce design rules, and execute automated visual quality assurance.

## End-to-End Execution Flow

```text
USER REQUEST
       │
       ▼
BUSINESS QUESTION
       │
       ▼
 DATA PROFILING
       │
  ┌────┴────┐
  ▼         ▼
Variable Types    Business Context
  │         │
  └────┬────┘
       ▼
 Analytical Intent
       │
       ▼
chart_selection.json
       │
       ▼
 Candidate Charts
       │
       ▼
chart_library.json
       │
       ▼
 Select Chart
       │
  ┌────┼─────────────┐
  ▼    ▼             ▼
Design Color       Axes
 Rules Rules       Rules
  │    │             │
  └────┼─────────────┘
       ▼
 Business Rules
       │
       ▼
 Accessibility
       │
       ▼
 Anti-patterns
       │
       ▼
 Generate Visualization
       │
       ▼
 Visualization QA
       │
  FAIL │  PASS
  ┌────┴────┐
  ▼         ▼
 Revise   Output
