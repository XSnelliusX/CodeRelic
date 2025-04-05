**CodeRelic - Technical Design Document (TDD)**

**Version:** 1.0
**Date:** 04/05/2025
**Status:** Draft
**Related PRS Version:** 1.0

**Table of Contents:**

1.  Introduction
    1.1 Purpose
    1.2 Scope
    1.3 Goals and Non-Goals
    1.4 Design Principles
2.  System Architecture Overview
    2.1 High-Level Architecture Diagram (Conceptual)
    2.2 Technology Stack Rationale
    2.3 Communication Patterns
3.  Component Design (Major Services)
    3.1 Frontend Service (UI)
    3.2 API Gateway
    3.3 Project Management Service
    3.4 Source Code Ingestion Service
    3.5 Parsing Service
    3.6 Static Analysis Service
    3.7 Semantic Modeling Service
    3.8 Transformation Service
    3.9 Code Generation Service
    3.10 Testing & Validation Service
    3.11 Reporting Service
    3.12 Knowledge Base Service
    3.13 Notification Service
4.  Data Management Design
    4.1 Relational Database Schema Overview (PostgreSQL)
    4.2 Object Storage Usage (S3-Compatible / MinIO)
    4.3 Cache Usage (Redis)
    4.4 Data Flow Summary
5.  Intermediate Representation (IR) Design
    5.1 Goals and Requirements
    5.2 Proposed Structure
    5.3 Key Information Represented
    5.4 Serialization
6.  Deployment Architecture
    6.1 Deployment Model (On-Premise Kubernetes)
    6.2 Containerization (Docker)
    6.3 Orchestration (Kubernetes)
    6.4 Configuration Management
    6.5 Deployment Artifacts (Helm Chart)
7.  Security Design
    7.1 Authentication
    7.2 Authorization (RBAC)
    7.3 Data Encryption (In Transit, At Rest)
    7.4 Secret Management
    7.5 Network Security Considerations
    7.6 Auditing and Logging
    7.7 Supply Chain Security
8.  Scalability and Resilience Design
    8.1 Horizontal Scaling Strategy
    8.2 Load Balancing
    8.3 Fault Tolerance and Recovery
    8.4 Monitoring and Health Checks
9.  Testing Strategy Overview
10. Design Decisions & Tradeoffs

---

**1. Introduction**

**1.1 Purpose**
This Technical Design Document (TDD) outlines the technical approach and high-level architecture for implementing CodeRelic Version 1.0, as defined in the Project Requirements Specification (PRS) v1.1. It details the system's components, their interactions, technology choices, data management strategies, deployment model, and key design considerations necessary to achieve the specified functional and non-functional requirements.

**1.2 Scope**
This document covers the design of the CodeRelic system, including its microservices, frontend, data stores, intermediate representation, security mechanisms, and deployment strategy, sufficient to guide the development team. It directly addresses the requirements outlined in PRS v1.1. Detailed API specifications, database schemas, and IR specifications will be maintained in separate, linked documents or artifacts (e.g., OpenAPI specs, Data Model Doc, IR Spec Doc).

**1.3 Goals and Non-Goals**
**Goals:**
*   Design a modular, scalable, and resilient system based on a microservices architecture.
*   Ensure the architecture inherently supports the primary requirement of on-premise deployment and data security (CST-01, CST-02).
*   Define a robust processing pipeline capable of supporting the initial target languages and extensible for future additions.
*   Specify a powerful Intermediate Representation (IR) suitable for semantic analysis and cross-language transformation.
*   Incorporate AI/ML capabilities effectively within the pipeline for semantic understanding and generation tasks.
*   Prioritize functional equivalence validation through design choices (e.g., dedicated testing service, differential testing support).
*   Design for maintainability, testability, and operability within a Kubernetes environment.

**Non-Goals (for Version 1.0):**
*   Designing a cloud-hosted SaaS offering.
*   Achieving fully automated code modernization without any need for human review or intervention.
*   Designing automated architectural refactoring capabilities (e.g., monolith decomposition).
*   Providing a detailed design for every possible legacy language execution environment needed for differential testing (these are external dependencies).
*   Specifying low-level implementation details of every algorithm within this document.

**1.4 Design Principles**
The design of CodeRelic adheres to the following principles:
*   **Modularity:** Employing a microservices architecture to isolate concerns, allow independent development and scaling, and facilitate technology diversity.
*   **Scalability:** Designing key processing services to scale horizontally to handle varying loads and large codebases.
*   **Security by Design:** Integrating security considerations from the outset, focusing on the on-premise deployment model, secure communication, data protection, and robust access control.
*   **Resilience:** Utilizing asynchronous communication patterns (message queues) and designing for fault tolerance to handle transient failures and ensure system stability.
*   **Testability:** Designing components with clear interfaces and separation of concerns to facilitate unit, integration, and system testing, with a specific focus on enabling functional equivalence validation.
*   **Maintainability:** Adhering to coding standards, promoting clear documentation, using automated CI/CD, and implementing effective logging and monitoring.
*   **Extensibility:** Structuring the system, particularly language-specific components (Parsing, Transformation, Generation), to allow for future expansion with new languages or rules.

**2. System Architecture Overview**

**2.1 High-Level Architecture Diagram (Conceptual)**

*(Conceptual Description - A formal diagram should accompany this)*

The architecture is composed of the following major parts communicating over defined interfaces:

1.  **User Interface (UI):** A web-based SPA (Frontend Service) accessed by users.
2.  **API Gateway:** The single entry point for all UI requests and potentially external API consumers. Handles routing, authentication, rate limiting.
3.  **Backend Microservices:** A collection of specialized services responsible for core logic:
    *   *Management/Orchestration:* Project Management, Notification Service.
    *   *Core Pipeline:* Source Code Ingestion, Parsing, Static Analysis, Semantic Modeling, Transformation, Code Generation.
    *   *Support:* Testing & Validation, Reporting, Knowledge Base.
4.  **Asynchronous Communication Bus:** A Message Queue (MQ - RabbitMQ/Kafka) decouples long-running tasks (parsing, analysis, etc.) and enables resilience.
5.  **Persistence Layer:**
    *   **Relational Database (PostgreSQL):** Stores structured metadata (projects, users, configurations, results summaries).
    *   **Object Storage (S3-Compatible/MinIO):** Stores large binary artifacts (source code, IRs, generated code, reports).
    *   **Cache (Redis):** Stores temporary session data, caches frequently accessed non-critical data.
6.  **External Dependencies:** Git Repositories, Local LLM Endpoints, Legacy Execution Environments (for testing).

**Flow:** User interacts with UI -> UI sends requests to API Gateway -> Gateway routes to appropriate backend service (e.g., Project Mgmt) -> For long tasks, service publishes message to MQ -> Worker service consumes message (e.g., Parsing Service), performs task, stores results (OS/DB), publishes completion message -> Notification Service pushes updates to UI via WebSocket/SSE. Services interact directly for synchronous queries where needed (e.g., fetching project config).

**2.2 Technology Stack Rationale**

The following technology stack is chosen to meet the requirements outlined in the PRS:

*   **Frontend:** **TypeScript** with **React** (or Vue/Angular - final choice TBD based on team expertise). Provides a robust framework for building interactive SPAs. TypeScript enhances maintainability and reduces runtime errors.
*   **API Gateway:** **Go**-based implementation (e.g., custom using standard libs) or a managed solution like **Kong/Traefik**. Go offers excellent performance and concurrency for request handling.
*   **Backend Microservices:** A mix of languages based on suitability:
    *   **Python (FastAPI/Django):** Primary choice for services heavily involved in AI/ML (Semantic Modeling, Code Generation/Documentation via LLMs, potentially Analysis) due to its rich ecosystem (spaCy, PyTorch/TF, Hugging Face libraries). Also suitable for general logic (Project Management, Reporting). FastAPI offers good performance and automatic API docs.
    *   **Go:** Preferred for high-throughput, I/O-bound, or concurrency-intensive services (API Gateway, Ingestion Service, Notification Service, potentially Transformation workers) due to goroutines and performance.
    *   **Rust:** Considered for performance-critical and correctness-critical components like **Parsing** and potentially parts of **Transformation**, leveraging its memory safety and speed, especially when using libraries like `tree-sitter`.
*   **AI/ML:** Python libraries (e.g., **spaCy, NLTK, scikit-learn, PyTorch/TensorFlow, Hugging Face Transformers**), **ANTLR/tree-sitter** for parsing, client libraries for interacting with **local/on-prem LLM APIs** (e.g., via Ollama interface or direct API).
*   **Databases:**
    *   **PostgreSQL:** Robust, open-source RDBMS for structured metadata, supporting transactions and complex queries. Well-suited for user/project data.
    *   **Redis:** High-performance in-memory cache for session management, rate limiting counters, temporary task states.
*   **Message Queue:** **RabbitMQ** (initially preferred for task queue semantics) or **Kafka** (if streaming/event sourcing becomes more relevant). Provides asynchronous communication and decoupling.
*   **Object Storage:** **MinIO**. Provides an S3-compatible API suitable for on-premise deployment, storing large artifacts efficiently.
*   **Infrastructure:** **Docker** for containerization, **Kubernetes** for orchestration, **Helm** for packaging/deployment.
*   **CI/CD:** **GitLab CI / GitHub Actions** (or Jenkins) for automated build, test, and deployment pipelines.

**2.3 Communication Patterns**

*   **UI <-> Backend:** Synchronous Request/Response via HTTPS RESTful API calls from the SPA to the API Gateway.
*   **Backend <-> UI (Notifications):** Asynchronous push via WebSockets or Server-Sent Events (SSE) originating from the Notification Service.
*   **API Gateway <-> Microservices:** Synchronous Request/Response, typically REST over HTTP/S or potentially gRPC for internal high-performance communication.
*   **Microservice <-> Microservice (Synchronous):** Used sparingly for immediate data needs (e.g., Transformation service querying Project Mgmt for configuration). REST/HTTP or gRPC.
*   **Microservice -> MQ -> Microservice (Asynchronous):** Primary pattern for triggering and managing long-running tasks (parsing, analysis, translation, testing). Services publish task messages; dedicated worker instances consume messages, execute tasks, and potentially publish result messages. This ensures decoupling and resilience.
*   **Microservices <-> PostgreSQL:** Standard database connections using language-specific clients (e.g., psycopg2/SQLAlchemy for Python, pgx for Go). Connection pooling will be used.
*   **Microservices <-> Redis:** Standard Redis client libraries.
*   **Microservices <-> Object Storage:** Standard S3 client libraries (e.g., boto3 for Python, AWS SDK for Go) targeting the MinIO endpoint.
*   **Microservices <-> LLM Endpoint:** HTTPS RESTful API calls based on the specific LLM's API definition.

**3. Component Design (Major Services)**

*(Details for each service based on blueprint and PRS)*

**3.1 Frontend Service (UI)**
*   **Responsibilities:** Provide the web-based user interface (FR-UI-*), handle user input, display data and results, manage client-side state, communicate with the API Gateway.
*   **Core Logic:** SPA rendering, form handling, state management (e.g., using Redux/Zustand/Context API), API client logic, WebSocket/SSE handling for notifications.
*   **API Contracts:** Consumes REST API exposed by API Gateway. Emits WebSocket/SSE connection requests.
*   **Data Storage:** None directly; relies on backend via API. Uses browser local storage/session storage for UI state/tokens.
*   **Key Libraries:** React/Vue/Angular, TypeScript, CSS Framework (e.g., Tailwind CSS), State Management library, API client (e.g., Axios).

**3.2 API Gateway**
*   **Responsibilities:** Single entry point for UI/external requests, routing to backend services, handling Authentication/Authorization (token validation), Rate Limiting, potentially basic request/response transformations, aggregating responses (if needed).
*   **Core Logic:** Request routing based on path/headers, AuthN/AuthZ middleware (validating JWTs/session tokens), Rate limiting logic (e.g., token bucket), forwarding requests to appropriate upstream microservices.
*   **API Contracts:** Exposes the public REST API (documented via OpenAPI). Consumes internal APIs of backend services (REST/gRPC). Interacts with Auth service/Redis for validation.
*   **Data Storage:** May use Redis for rate limiting counters or session lookup.
*   **Key Libraries:** Go standard libraries (net/http), routing library (e.g., Gin, Chi), JWT validation library, potentially Kong/Traefik configuration.

**3.3 Project Management Service**
*   **Responsibilities:** Manage project lifecycle (CRUD operations - FR-PM-*), store/retrieve project configurations, track project status, manage user associations with projects.
*   **Core Logic:** REST API handlers for project operations, data validation, interaction with PostgreSQL for storing/retrieving project metadata and configurations. Publishes messages to MQ to trigger pipeline stages (e.g., "Start Parsing").
*   **API Contracts:** Exposes REST API for project management. Interacts with PostgreSQL. Publishes messages to MQ.
*   **Data Storage:** Primarily PostgreSQL (users, projects, project_configurations tables).
*   **Key Libraries:** Python/FastAPI, SQLAlchemy/sqlmodel (ORM), Pydantic (validation), MQ client library (e.g., Pika/Kombu).

**3.4 Source Code Ingestion Service**
*   **Responsibilities:** Retrieve source code from configured sources (Git, Upload - FR-ING-*), store it in Object Storage, notify relevant services upon completion.
*   **Core Logic:** Handles Git clone/pull operations (using credentials from secure storage), processes uploaded archives, interacts with Object Storage to save code artifacts, publishes "Code Ingested" message to MQ containing artifact location. Error handling for source access/processing.
*   **API Contracts:** Consumes messages from MQ (e.g., "Ingest Code for Project X"). Interacts with Git (CLI or library), Object Storage (S3 API). Publishes messages to MQ. May need access to secrets management for Git credentials.
*   **Data Storage:** Writes source code artifacts to Object Storage. May update status in PostgreSQL via Project Mgmt API or direct DB access (less ideal).
*   **Key Libraries:** Go/Python, Git library/CLI wrapper, S3 client library, MQ client library, archive processing libraries (zip, tarfile).

**3.5 Parsing Service**
*   **Responsibilities:** Parse source code for supported languages (FR-PARSE-*), generate initial ASTs or parse trees, handle syntax errors gracefully. Designed for language pluggability.
*   **Core Logic:** Consumes "Parse Code" messages from MQ. Retrieves source code from Object Storage. Invokes the appropriate language-specific parser (ANTLR-generated, tree-sitter based, custom). Stores the resulting AST/parse tree (e.g., as JSON/binary) in Object Storage. Reports syntax errors. Publishes "Code Parsed" message. May run as multiple instances/workers, potentially specialized per language.
*   **API Contracts:** Consumes messages from MQ. Interacts with Object Storage. Publishes messages to MQ.
*   **Data Storage:** Reads source code from Object Storage, writes AST/parse trees to Object Storage.
*   **Key Libraries:** Rust/Python/Go, ANTLR runtime, tree-sitter library, MQ client library, S3 client library, serialization library (e.g., Serde for Rust, json/pickle for Python).

**3.6 Static Analysis Service**
*   **Responsibilities:** Analyze ASTs/parse trees (FR-ANLS-*), build Control Flow Graphs (CFGs), perform basic Data Flow Analysis, identify code structures, dependencies, patterns, and metrics (e.g., complexity).
*   **Core Logic:** Consumes "Analyze Code" messages. Retrieves ASTs from Object Storage. Performs analysis using graph algorithms, pattern matching (rule-based and potentially ML-based). Enriches or generates data structures representing analysis results (e.g., annotated AST, CFG representation). Stores results in Object Storage or potentially updates IR. Publishes "Analysis Complete" message.
*   **API Contracts:** Consumes messages from MQ. Interacts with Object Storage. Publishes messages to MQ.
*   **Data Storage:** Reads ASTs from Object Storage, writes analysis results/enriched structures to Object Storage.
*   **Key Libraries:** Python/Go/Rust, Graph libraries (e.g., NetworkX for Python), potentially ML libraries (scikit-learn), MQ client, S3 client.

**3.7 Semantic Modeling Service**
*   **Responsibilities:** Build the rich, language-agnostic Intermediate Representation (IR) (FR-IR-*). Integrate parsing and analysis results, infer types and semantic information, potentially leverage NLP/ML for deeper understanding.
*   **Core Logic:** Consumes "Model Semantics" messages. Retrieves ASTs and analysis results from Object Storage. Constructs the core IR graph structure, normalizing language-specific constructs, resolving symbols, inferring types, potentially using ML models (embeddings, GNNs) trained on code/comments to capture intent. Stores the final IR in Object Storage. Publishes "IR Generated" message.
*   **API Contracts:** Consumes messages from MQ. Interacts with Object Storage. May interact with LLM endpoints for semantic enrichment. Publishes messages to MQ.
*   **Data Storage:** Reads ASTs/Analysis results from Object Storage, writes final IR to Object Storage.
*   **Key Libraries:** Python (likely candidate due to AI/ML), Graph libraries, ML libraries (PyTorch/TF, spaCy, Hugging Face), MQ client, S3 client, LLM client library.

**3.8 Transformation Service**
*   **Responsibilities:** Apply modernization rules, version-specific upgrades (FR-VER-*), language translation logic, and user customizations (FR-CUS-*) to the IR (FR-TRANS-*).
*   **Core Logic:** Consumes "Transform IR" messages. Retrieves IR from Object Storage. Fetches project configuration (target language, styles, rules) potentially via Project Mgmt API or cached config. Queries Knowledge Base Service for applicable rules. Traverses and modifies the IR based on rules and configuration. Stores the transformed IR in Object Storage. Publishes "IR Transformed" message.
*   **API Contracts:** Consumes messages from MQ. Interacts with Object Storage. Interacts with Knowledge Base Service API. May interact with Project Management Service API. Publishes messages to MQ.
*   **Data Storage:** Reads input IR from Object Storage, writes transformed IR to Object Storage.
*   **Key Libraries:** Python/Go/Rust, Graph manipulation libraries, MQ client, S3 client, API client libraries.

**3.9 Code Generation Service**
*   **Responsibilities:** Translate the transformed IR into target language syntax (FR-GEN-*), incorporate comments/documentation (FR-DOC-*), apply formatting/linting (FR-CUS-002).
*   **Core Logic:** Consumes "Generate Code" messages. Retrieves transformed IR from Object Storage. Uses target-language-specific generators (template-based, rule-based, potentially LLM-assisted) to produce source code strings. Integrates comments/docs (potentially calling LLM service). Applies configured linters/formatters. Stores generated code in Object Storage. Publishes "Code Generated" message.
*   **API Contracts:** Consumes messages from MQ. Interacts with Object Storage. May interact with LLM endpoints. Publishes messages to MQ. May invoke external linter/formatter tools.
*   **Data Storage:** Reads transformed IR from Object Storage, writes generated code to Object Storage.
*   **Key Libraries:** Python/Go/Rust, Template engines (e.g., Jinja2), LLM client library, MQ client, S3 client, subprocess execution (for formatters).

**3.10 Testing & Validation Service**
*   **Responsibilities:** Generate test stubs (FR-TEST-*), orchestrate functional equivalence testing (differential testing) (FR-FEQ-*), run static analysis on generated code, report results.
*   **Core Logic:** Consumes "Validate Code" messages. Retrieves generated code (and potentially original code) from Object Storage. Generates test stubs based on code structure. Orchestrates differential testing: sets up configured legacy/modern execution environments (likely as temporary K8s pods/containers based on user-provided images/configs), executes code with specified inputs, compares outputs based on defined criteria (FR-FEQ-005.1). Runs configured linters/SAST tools. Stores detailed results in Object Storage and summary results/status in PostgreSQL. Publishes "Validation Complete" message.
*   **API Contracts:** Consumes messages from MQ. Interacts with Object Storage. Interacts with PostgreSQL (for results summary). Needs K8s API access to manage test execution pods. May invoke external tools (linters, test frameworks, emulators). Publishes messages to MQ.
*   **Data Storage:** Reads code from OS, writes test stubs to OS, writes detailed test reports/logs to OS, writes summary results to PostgreSQL (validation_results table).
*   **Key Libraries:** Python/Go, Test framework integration libraries (pytest, go test), K8s client library, S3 client, DB client, MQ client, subprocess execution.

**3.11 Reporting Service**
*   **Responsibilities:** Generate side-by-side diff reports (FR-DIFF-*), analysis summaries, validation summaries based on data from various stages.
*   **Core Logic:** Consumes "Generate Report" messages or responds to API requests. Retrieves original code, translated code, analysis metadata, validation results from Object Storage and/or PostgreSQL. Generates diff views (using diffing libraries), summary reports (potentially Markdown or HTML). Stores generated reports in Object Storage.
*   **API Contracts:** Consumes messages from MQ or serves API requests (via Gateway). Interacts with Object Storage and PostgreSQL. Publishes messages to MQ ("Report Ready") or returns report data synchronously.
*   **Data Storage:** Reads data from OS and PostgreSQL. Writes generated reports to Object Storage.
*   **Key Libraries:** Python/Go, Diffing libraries (e.g., difflib), Template engines (for report formatting), S3 client, DB client, MQ client (optional).

**3.12 Knowledge Base Service**
*   **Responsibilities:** Store and serve language migration rules, deprecated API mappings, common transformation patterns, potentially fine-tuned models or rule configurations.
*   **Core Logic:** Provides an internal API (likely REST) for other services (mainly Transformation) to query rules based on source/target language, version, code constructs. May have its own persistent storage (DB or files) for the knowledge base content. Provides mechanisms for updating the knowledge base.
*   **API Contracts:** Exposes internal REST API for querying rules. May have internal storage interface.
*   **Data Storage:** Potentially its own database (PostgreSQL subset or dedicated store) or reads from structured files in Object Storage / mounted volume.
*   **Key Libraries:** Python/Go/Rust, Web framework (FastAPI/Gin), DB client/file access libraries.

**3.13 Notification Service**
*   **Responsibilities:** Manage real-time communication (status updates, alerts) to connected UI clients (FR-UI-005).
*   **Core Logic:** Consumes internal notification messages from MQ (published by various pipeline services upon completion/failure). Manages active WebSocket/SSE connections from UI clients. Pushes relevant updates to appropriate clients based on user session or project subscription.
*   **API Contracts:** Consumes messages from MQ. Manages WebSocket/SSE connections. May interact with Redis/DB to map connections to users/projects.
*   **Data Storage:** Potentially uses Redis to track active connections and user mappings.
*   **Key Libraries:** Go/Python, WebSocket library (e.g., Gorilla WebSocket for Go, FastAPI WebSockets), MQ client, Redis client.

**4. Data Management Design**

**4.1 Relational Database Schema Overview (PostgreSQL)**
*   **Purpose:** Stores structured metadata, configuration, status, user information, and result summaries.
*   **Key Tables (Conceptual - see PRS Section 11 for details):** `users`, `projects`, `project_configurations`, `source_artifacts` (metadata), `intermediate_representations` (metadata), `translation_outputs` (metadata), `analysis_reports` (metadata/summary), `test_scaffolds` (metadata), `validation_results` (summary), `knowledge_base_rules`.
*   **Relationships:** Foreign keys link related entities (e.g., `project_configurations` to `projects`, `translation_outputs` to `projects`).
*   **Access:** Accessed primarily by Project Management, Reporting, and Testing & Validation services.
*   **Tooling:** Schema migrations managed using tools like Alembic (Python).
*   **Note:** A detailed Data Model Document / ERD will provide the final schema.

**4.2 Object Storage Usage (S3-Compatible / MinIO)**
*   **Purpose:** Stores large, potentially unstructured artifacts generated during the pipeline.
*   **Data Stored:** Original source code archives/files, ASTs/parse trees, Intermediate Representations (IRs), generated source code, generated documentation files, detailed analysis reports, diff reports, test logs, test output comparisons, test scaffolding files.
*   **Structure (Conceptual):** A single bucket (e.g., `coderedlic-data`) with a logical structure using prefixes/folders:
    *   `projects/<project_id>/source/<commit_hash_or_upload_id>/...`
    *   `projects/<project_id>/artifacts/<run_id>/ast/...`
    *   `projects/<project_id>/artifacts/<run_id>/ir/raw/...`
    *   `projects/<project_id>/artifacts/<run_id>/ir/transformed/...`
    *   `projects/<project_id>/artifacts/<run_id>/output/<target_lang>/...`
    *   `projects/<project_id>/artifacts/<run_id>/reports/...`
    *   `projects/<project_id>/artifacts/<run_id>/tests/stubs/...`
    *   `projects/<project_id>/artifacts/<run_id>/tests/validation/...`
*   **Access:** Accessed by most backend services for reading/writing artifacts.

**4.3 Cache Usage (Redis)**
*   **Purpose:** Stores short-lived, frequently accessed data to improve performance and manage state.
*   **Data Stored:** User session information, API Gateway rate limiting counters, temporary locks for distributed operations (if needed), potentially cached configuration snippets, active WebSocket/SSE connection mappings.
*   **Access:** Accessed by API Gateway, Notification Service, potentially others needing session data or distributed coordination.

**4.4 Data Flow Summary**
1.  User initiates project -> Config stored in PostgreSQL (via Proj Mgmt).
2.  Code linked/uploaded -> Stored in Object Storage (via Ingestion).
3.  Pipeline starts -> Each stage reads input (code/AST/IR) from OS, writes output (AST/IR/code/report) to OS. Status updates may go to PostgreSQL.
4.  Notifications -> Sent via MQ -> Notification Service -> WebSocket/SSE to UI.
5.  User views results -> UI queries API Gateway -> Services retrieve metadata from PostgreSQL and potentially artifact links/summaries from OS -> UI may fetch large artifacts directly from OS via signed URLs (if implemented) or proxied via backend.

**5. Intermediate Representation (IR) Design**

**5.1 Goals and Requirements**
*   **Language-Agnostic:** Represent concepts from COBOL, Perl, Python 2, Python 3, Go, TypeScript, Rust.
*   **Richly Semantic:** Capture AST structure, control flow, data flow, types, dependencies, scope, potentially higher-level patterns and inferred semantics (FR-IR-*).
*   **Transformable:** Easy to traverse, query, and modify by the Transformation service.
*   **Serializable:** Efficiently stored (Object Storage) and passed between services (FR-IR-005).

**5.2 Proposed Structure**
*   **Graph-Based:** A custom Code Property Graph (CPG)-like structure is proposed. This combines aspects of:
    *   **Abstract Syntax Tree (AST):** Nodes representing code syntax elements (variables, expressions, statements, functions, classes, etc.).
    *   **Control Flow Graph (CFG):** Edges representing the flow of execution between basic blocks or statements.
    *   **Data Flow Graph (DFG):** Edges representing the flow of data (e.g., where a variable defined is used).
    *   **Symbol Table / Scope Information:** Linking identifiers to their declarations and scopes.
*   **Nodes:** Represent code elements, types, files, methods, etc., with properties (e.g., code snippet, line number, type, name).
*   **Edges:** Represent relationships (e.g., `CONTAINS` for hierarchy, `FLOWS_TO` for control flow, `REACHES` for data flow, `CALLS` for function calls, `IMPORTS`, `INHERITS`).
*   **Annotations:** Nodes and edges can hold additional metadata derived from static analysis or ML models (e.g., inferred type confidence, semantic role, link to original legacy pattern).

**5.3 Key Information Represented**
*   Files, Modules, Packages
*   Functions, Methods, Classes, Interfaces, Structures
*   Variables, Parameters, Fields (with scope)
*   Statements (Assignments, Loops, Conditionals, Calls, Returns, Exception Handling)
*   Expressions (Literals, Operators, References)
*   Type Information (Explicit from source, Inferred from analysis)
*   Dependencies (Internal and External)
*   Control Flow Paths
*   Data Provenance (limited)
*   Comments and source position mapping.

**5.4 Serialization**
*   **Format:** **JSON** (human-readable, widely supported) or potentially a more compact binary format like **Protocol Buffers** (efficient, schema-defined) for storage and inter-service communication. The choice depends on performance vs. debuggability tradeoffs.
*   **Storage:** Serialized IRs stored as files in Object Storage.

**Note:** A separate, detailed IR Specification document will define the precise node types, edge types, properties, and serialization format.

**6. Deployment Architecture**

**6.1 Deployment Model (On-Premise Kubernetes)**
*   CodeRelic is designed exclusively for deployment within the customer's Kubernetes cluster (on-premise data center or private cloud VPC), satisfying CST-01 and NFR-PORT-001.

**6.2 Containerization (Docker)**
*   All CodeRelic microservices (frontend and backend) will be packaged as Docker images (FR-ONPREM-001). Dockerfiles will follow best practices (multi-stage builds, minimal base images, non-root users - NFR-SEC-006).

**6.3 Orchestration (Kubernetes)**
*   Kubernetes is the required orchestrator (CST-03). CodeRelic components will be deployed as K8s Deployments, Services, potentially StatefulSets (for DB/MQ if deployed within cluster), managed via Helm.
*   Leverages K8s features: Service Discovery, Load Balancing, Secrets Management, ConfigMaps, Persistent Volumes (for DB/MQ data if co-deployed), Horizontal Pod Autoscaling (HPA) (NFR-SCAL-003), Health Checks (Liveness/Readiness Probes) (NFR-MAIN-006).

**6.4 Configuration Management**
*   **Helm Values:** Primary mechanism for customer configuration during deployment (database URLs, MQ endpoints, object storage credentials, resource limits, LLM endpoint, license key, etc.) (FR-ONPREM-002, FR-ONPREM-003, FR-ONPREM-004).
*   **Kubernetes ConfigMaps:** Used internally to propagate non-sensitive configuration to pods.
*   **Kubernetes Secrets:** Used for all sensitive data (passwords, API keys, tokens, SSH keys) (NFR-SEC-003).

**6.5 Deployment Artifacts (Helm Chart)**
*   A comprehensive Helm chart will be provided to manage the deployment, configuration, and lifecycle (install, upgrade, uninstall) of all CodeRelic services and their K8s resources (NFR-PORT-004).

**7. Security Design**

**7.1 Authentication**
*   **Method:** Supports external IdP integration via SAML 2.0 or OIDC (FR-SEC-002). Local username/password authentication available as fallback/alternative.
*   **Flow:** User authenticates via UI -> Redirect to IdP (if external) -> IdP redirects back with assertion/token -> Backend validates token/assertion, establishes session (e.g., issues JWT) -> UI uses JWT for subsequent API calls via API Gateway.
*   **Token:** JWTs preferred for stateless authentication between UI and API Gateway. Internal service-to-service auth may use simpler mechanisms if network is trusted, or short-lived tokens.

**7.2 Authorization (RBAC)**
*   **Model:** Role-Based Access Control (FR-SEC-003). Initial roles: `Administrator`, `User` (FR-SEC-004).
*   **Enforcement:** Likely enforced at the API Gateway based on roles/permissions present in the validated JWT claims. Finer-grained checks may occur within services if needed.

**7.3 Data Encryption**
*   **In Transit:** TLS 1.2+ mandatory for all external communication (UI, API Gateway) and recommended for all internal communication (service-to-service, service-to-DB/MQ/OS) (EI-COM-*, NFR-SEC-002). Configuration via Helm chart.
*   **At Rest:** Encryption of data volumes (PostgreSQL data, K8s Secrets) and object storage buckets relies on underlying infrastructure capabilities (e.g., LUKS, AWS EBS encryption, S3 bucket encryption) and must be enabled/configured by the customer administrator (NFR-SEC-004).

**7.4 Secret Management**
*   All secrets (DB passwords, Git tokens, LLM keys, license keys, etc.) managed via Kubernetes Secrets, populated via Helm values during installation (NFR-SEC-003). Secrets mounted into pods as environment variables or files.

**7.5 Network Security Considerations**
*   Deployment within customer's secure network perimeter.
*   Use of Kubernetes Network Policies is recommended (but optional for customer to configure) to restrict traffic flow between pods/namespaces for enhanced segmentation.
*   API Gateway acts as the primary ingress point, limiting direct exposure of backend services.

**7.6 Auditing and Logging**
*   Structured logging (JSON) from all services (NFR-MAIN-005).
*   Specific audit logs for security-sensitive events (logins, project deletion, user management) generated (NFR-SEC-008). Logs collected via standard K8s logging agents (e.g., Fluentd, Logstash).

**7.7 Supply Chain Security**
*   Regular dependency scanning in CI pipeline (NFR-SEC-005).
*   Use of minimal, scanned base container images (NFR-SEC-006).
*   Software Bill of Materials (SBOM) generation (OR-LIC-002).

**8. Scalability and Resilience Design**

**8.1 Horizontal Scaling Strategy**
*   Stateless processing services (Parsing, Analysis, Semantic Modeling, Transformation, Code Generation, Testing workers) designed as K8s Deployments, scalable via HPA based on MQ queue depth or CPU/Memory usage (NFR-SCAL-003).
*   Frontend Service and API Gateway Deployments can also be scaled horizontally based on request load/CPU.
*   Stateful components (PostgreSQL, RabbitMQ/Kafka, MinIO) rely on their respective clustering/scaling mechanisms, potentially managed outside K8s or via Operators if deployed within.

**8.2 Load Balancing**
*   Kubernetes Services provide internal load balancing across pods of a Deployment.
*   Kubernetes Ingress controller manages external load balancing to the API Gateway.

**8.3 Fault Tolerance and Recovery**
*   **Asynchronous Processing:** MQ decouples services, preventing cascading failures and allowing retries (NFR-REL-002).
*   **Retries:** Automatic retries with exponential backoff for transient errors in MQ consumers.
*   **Dead-Letter Queues (DLQ):** Failed messages moved to DLQs for investigation (NFR-REL-003).
*   **Pod Health Checks:** Kubernetes restarts unhealthy pods based on liveness/readiness probes (NFR-MAIN-006).
*   **Database/Storage:** Reliability depends on the chosen external DB/MQ/Storage deployment and its high-availability configuration. Backup/restore procedures documented (OR-DATA-003).

**8.4 Monitoring and Health Checks**
*   Standardized `/health/live` and `/health/ready` endpoints on all services for K8s probes (NFR-MAIN-006).
*   Exposure of key metrics (e.g., request latency, queue depths, processing times, error rates) via Prometheus-compatible endpoints (e.g., using client libraries) for scraping by monitoring systems.
*   Structured logging enables log-based monitoring and alerting (NFR-MAIN-005).

**9. Testing Strategy Overview**
A comprehensive testing strategy is crucial for CodeRelic's success, particularly regarding functional equivalence. The high-level approach includes:
*   **Unit Tests:** Developers write tests for individual functions/classes in each microservice (NFR-MAIN-003).
*   **Integration Tests:** Verify interactions between services (API calls, MQ messages, DB interactions).
*   **Component Tests:** Test the functionality of a single microservice in isolation (mocking external dependencies).
*   **System/End-to-End Tests:** Test complete workflows via the UI or API Gateway (e.g., create project, upload code, translate, view diff, export).
*   **Functional Equivalence / Differential Testing:** Dedicated focus using the Testing & Validation service to compare legacy and translated code outputs (FR-FEQ-*). This requires careful setup and environment configuration.
*   **Performance & Load Tests:** Simulate concurrent users and jobs to validate NFRs (NFR-PERF-*, NFR-SCAL-*).
*   **Security Tests:** Dependency scanning, container scanning, potentially DAST/SAST and penetration testing.
*   **Installation Tests:** Verify deployment via Helm chart in target environments.

**Note:** A separate, detailed **Testing Plan** document will elaborate on methodologies, tools, environments, responsibilities, and specific test cases for each level/type.

**10. Design Decisions & Tradeoffs**

*   **Microservices:**
    *   *Decision:* Adopt microservices architecture.
    *   *Rationale:* Scalability, modularity, technological flexibility (PRS NFRs).
    *   *Tradeoff:* Increased operational complexity, potential network latency, distributed transaction challenges.
*   **On-Premise First:**
    *   *Decision:* Mandate on-premise deployment for core processing.
    *   *Rationale:* Security requirement (CST-01, CST-02).
    *   *Tradeoff:* Higher barrier to entry for customers (infra needed), more complex installation/maintenance compared to SaaS.
*   **Asynchronous Pipeline:**
    *   *Decision:* Use Message Queue for core pipeline orchestration.
    *   *Rationale:* Resilience, scalability, decoupling (NFR-REL-002, NFR-SCAL-003).
    *   *Tradeoff:* Eventual consistency, increased complexity in debugging workflows.
*   **Intermediate Representation (IR):**
    *   *Decision:* Develop a custom, graph-based IR.
    *   *Rationale:* Need for rich semantic representation across diverse languages, control over transformation capabilities.
    *   *Tradeoff:* Significant design and implementation effort compared to using an existing (potentially less suitable) IR.
*   **Technology Mix (Python/Go/Rust):**
    *   *Decision:* Use multiple backend languages.
    *   *Rationale:* Leverage best language features for specific tasks (AI/ML vs. Concurrency vs. Safety/Performance).
    *   *Tradeoff:* Requires broader team expertise, potentially more complex build/CI setup.
*   **Differential Testing:**
    *   *Decision:* Include differential testing orchestration.
    *   *Rationale:* Critical for functional equivalence validation (FR-FEQ-*).
    *   *Tradeoff:* Complex to implement reliably, heavy dependency on customer configuring legacy execution environments. Accuracy depends heavily on input quality and equivalence definition.
*   **LLM Integration (Local Endpoint Focus):**
    *   *Decision:* Require support for local LLM endpoints.
    *   *Rationale:* Data privacy/security constraint (CST-05).
    *   *Tradeoff:* Customer burden for hosting/managing LLMs, potential performance variations based on local model.