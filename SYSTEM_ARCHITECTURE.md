# 🏛️ System Architecture Documentation

## Universal Constrained Synthetic Dataset Generator

This document provides a comprehensive technical overview of the system architecture, component interactions, data flows, and verification lifecycle for the **Synthetic Data Studio**.

---

## 1. High-Level Architecture Overview

The system employs a 5-tier modular architecture designed for high throughput, strict schema adherence, and zero-hallucination synthetic data delivery:

```mermaid
flowchart TD
    subgraph Tier1["Tier 1: Presentation & Interaction (Gradio Web UI)"]
        UI_Schema["Schema Ingestion\n• File Upload (.json, .csv, .txt, .md)\n• Interactive Code Editor"]
        UI_Controls["Generation Controls\n• Row Count (1 – 10,000)\n• Engine Mode Selector\n• Seed & Refinement Toggles"]
        UI_Preview["Live Schema Inferred Preview"]
        UI_Studio["Interactive Data Studio\n• Paginated Browser & Filter\n• Column Profiling & Histograms\n• Real-Time Quality Report"]
        UI_DL["Quality Gate Protected Downloads\n(CSV · JSON · Excel)"]
    end

    subgraph Tier2["Tier 2: Ingestion & Orchestration Engine"]
        Parser["Dynamic Schema Parser\n• Normalizes Draft 2020-12 & Draft-07\n• Resolves $defs & Nested Structures\n• Meta-Schema vs. Domain Detection"]
        SpecGen["Execution Specification Generator\n• Constraint Extraction (Bounds, Enums, RegEx)\n• Statistical Distribution Mapping\n• Cross-Column Relational Rules"]
    end

    subgraph Tier3["Tier 3: Dual Generation Engines"]
        direction TB
        subgraph ConstraintEngine["⚡ High-Speed Constraint Engine"]
            Catalog["Enterprise Database Catalog (50 Specs)"]
            StatSampler["Statistical Sampler (Normal, Log-normal, Uniform)"]
            FakerProvider["Faker Identity & Temporal Provider"]
        end

        subgraph LLMEngine["🦙 Local LLM Engine (Ollama)"]
            PromptEng["Prompt Engine (7 Strict Rules Enforced)"]
            OllamaDaemon["Ollama REST Daemon (:11434)\n(Qwen 2.5 14B / LLaMA 3.2)"]
            StreamParser["JSON Response Parser & Cleaner"]
        end
    end

    subgraph Tier4["Tier 4: Verification & Automated Self-Healing"]
        direction TB
        Verifier["Verification Suite v2 (16+ Tests)\n• Identifier Uniqueness & Snake_case\n• Type-Compatible Constraints\n• Min/Max and Min_length/Max_length\n• JSON Arrays & Objects Conformance\n• Meaningful Field Descriptions\n• Valid Statistical Distributions\n• Strict JSON Schema Validation\n• Domain Business Rules & Sentiment"]
        
        Refiner["Automated Self-Correction Engine\n• Inverted Bound Swapping\n• Categorical Enum Clamping\n• Type Incompatibility Stripping\n• Review-Rating Alignment\n• Log-Normal Income Skewing"]
    end

    subgraph Tier5["Tier 5: Storage & Serialization"]
        MemoryDF["In-Memory Pandas DataFrame Engine"]
        CSVSerializer["CSV Exporter"]
        JSONSerializer["JSON Records Exporter"]
        ExcelSerializer["Excel (.xlsx) Exporter"]
        AuditReport["Audit JSON & Log Generator"]
    end

    %% Wiring
    UI_Schema --> Parser
    UI_Controls --> SpecGen
    Parser --> SpecGen
    SpecGen --> UI_Preview

    SpecGen -->|Constraint Mode / Offline| ConstraintEngine
    SpecGen -->|LLM Mode / Ollama Online| LLMEngine

    Catalog --> StatSampler --> FakerProvider --> RawData["Raw Generated Batch"]
    PromptEng --> OllamaDaemon --> StreamParser --> RawData

    RawData --> Verifier
    Verifier -->|Violations Found| Refiner
    Refiner -->|Healed Records| Verifier
    Verifier -->|Quality Gate Passed: 100/100| MemoryDF

    MemoryDF --> UI_Studio
    MemoryDF --> CSVSerializer --> UI_DL
    MemoryDF --> JSONSerializer --> UI_DL
    MemoryDF --> ExcelSerializer --> UI_DL
    AuditReport --> UI_DL
```

---

## 2. Sequence Diagram: Generation & Quality Gate Flow

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Gradio Web UI
    participant Parser as Schema Analyzer
    participant Router as Engine Router
    participant CE as Constraint Engine
    participant LLM as Ollama LLM (Qwen 14B)
    participant Verifier as Verifier Suite v2
    participant Refiner as Self-Healing Engine
    participant Storage as Export Engine

    User->>UI: Uploads schema & selects row count
    UI->>Parser: Parse & analyze schema
    Parser-->>UI: Return detected columns & inferred types
    UI->>User: Display Live Schema Preview

    User->>UI: Clicks "Generate dataset"
    UI->>Router: Route generation task
    
    alt Generation Mode = Ollama LLM & Online
        Router->>LLM: Stream batch prompt (10 rows/batch)
        LLM-->>Router: Return raw JSON records
    else Generation Mode = Constraint Engine or Offline
        Router->>CE: Sample vectorized batch records
        CE-->>Router: Return synthetic records
    end

    Router->>Verifier: Run 16+ programmatic quality tests
    
    alt Any Verification Check Fails
        Verifier->>Refiner: Trigger automated self-correction
        Refiner->>Refiner: Swap inverted bounds, clamp enums, strip invalid types
        Refiner->>Verifier: Re-run verification on healed dataset
    end

    Verifier->>UI: Report Final Score (100.0/100.0 PASS)
    Verifier->>Storage: Serialize to CSV, JSON, and Excel
    Storage-->>UI: Unlock download buttons
    UI-->>User: Present interactive data studio & download links
```

---

## 3. Subsystem Specifications

### 3.1. Ingestion & Analysis Engine
* **Draft Compatibility**: Supports JSON Schema Draft-07, 2020-12, raw JSON column lists, and CSV headers.
* **Schema Topology Detection**:
  * **Domain Dataset**: User specifies actual data entities (e.g. `customer_id`, `age`, `annual_income`, `rating`, `review`).
  * **Field Configuration Dataset**: User specifies database column metadata (e.g. `name`, `type`, `description`, `min`, `max`, `allowed_values`, `weights`, `precision`, `items`).

### 3.2. Generation Engine Subsystems
* **High-Speed Constraint Engine**:
  * Vectorized sampling via NumPy (`lognormal`, `normal`, `uniform`).
  * Seed-reproducible Faker providers.
  * 50 built-in enterprise database specifications for instant field config synthesis.
* **Local LLM Engine (Ollama)**:
  * REST integration with `http://localhost:11434/api/generate`.
  * Enforces prompt-level guardrails on `qwen2.5:14b` / `llama3.2`.
  * Markdown stripping and resilient JSON stream extraction.

### 3.3. Quality Gate & Self-Correction Pipeline
* **Fail-Closed Gate**: Downloads are strictly locked if any test fails or score < 90.
* **Verification Checks**:
  1. `row_count`: Exact row count preservation.
  2. `column_names`: Exact column parity.
  3. `data_types`: Strict typing conformance.
  4. `numerical_ranges`: Boundary check ($min \le x \le max$).
  5. `categorical_values`: Universal enum set validation.
  6. `duplicate_rows`: Zero full-row duplicate guarantee.
  7. `identifier_uniqueness`: 100% unique primary keys.
  8. `categorical_diversity`: Shannon entropy threshold testing.
  9. `meaningful_field_identifiers`: Enforces snake_case identifiers, rejects human names with spaces.
  10. `type_compatible_constraints`: Eliminates numeric bounds from non-numeric types.
  11. `min_less_equal_max`: Guarantees $min \le max$.
  12. `min_length_less_equal_max_length`: Guarantees $min\_length \le max\_length$.
  13. `correct_arrays_and_objects`: Enforces valid JSON arrays and objects.
  14. `meaningful_descriptions`: Rejects single-word placeholders.
  15. `valid_distributions_and_weights`: Restricts distributions to valid options.
  16. `json_schema_validation`: Complete schema property validation.

---

## 4. File Mapping

| Component | File / Location | Responsibility |
| :--- | :--- | :--- |
| **Gradio Web Dashboard** | `dataset_generator_dashboard (1).ipynb` (Cell 25) | User interaction, profiling, data browser, download locking |
| **Verification Suite** | `dataset_generator_dashboard (1).ipynb` (Cell 21) | Comprehensive verification and scoring |
| **Self-Correction Engine** | `dataset_generator_dashboard (1).ipynb` (Cell 23) | Multi-pass programmatic repair pipeline |
| **Generation Engines** | `dataset_generator_dashboard (1).ipynb` (Cell 18) | Dual constraint & Ollama generation |
| **Sample Field Config Schema** | `dataset_generator.schema.json` | 18-column universal field configuration schema |
| **Sample Customer Schema** | `customer_dataset.schema.json` | Domain customer demographics and review schema |
