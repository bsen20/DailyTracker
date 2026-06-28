Act as both a rigorous Senior Engineering Interviewer and an expert Candidate participating in a System Design interview in June 2026.

Your task is to generate a comprehensive transcript of a system design interview. Do not dump the entire design at once. The output must read like an actual transcript of a dynamic, back-and-forth conversation as the system architecture organically evolves.

**Problem:** [Design Scalable OTT Streaming Platform like - 'Netflix' , 'Amazon Prime']

### Interview Simulation Guidelines:

1. **Realistic Dialogue Flow:** The Interviewer must act as a senior technical leader—asking probing questions, introducing constraints, pushing back on assumptions, and challenging the Candidate. The Candidate must answer thoughtfully, focusing on the "Why", "What", and "When", and robustly defending their trade-offs.
2. **2026 Senior/Architect Standards:** Grade and guide the candidate based on modern expectations:
   - **Operational Thinking:** The candidate must explicitly discuss failure modes, graceful degradation, observability, and rollback paths.
   - **Cost Reasoning:** Discuss the financial and resource trade-offs of architectural choices (e.g., horizontal scaling vs. optimizing efficiency).
   - **Distributed Systems Realities:** Address eventual vs. strong consistency, network partitions, idempotency, and handling thundering herds.
3. **Scope Constraints (No Implementation Code):** Keep the focus strictly on High-Level Design (HLD) and specific Low-Level Design (LLD). Exclude all implementation code or business logic. Limit LLD to:
   - API Contracts (REST/gRPC structures, request/response payloads).
   - Database Schema Design (Tables/Collections, partition keys, indexing strategies).
4. **Visual Progression (Mermaid Diagrams):** Use Mermaid.js diagrams heavily to illustrate the system at different stages.
   - Provide a simple HLD diagram for the "Happy Path."
   - Provide updated, highly detailed diagrams as the system scales (e.g., adding load balancers, caching layers, message queues, and sharded databases).

### Required Transcript Structure:

**Phase 1: Clarification & Scope**

- The Candidate asks clarifying questions regarding scale, read/write ratios, and specific user personas.
- The Interviewer defines the exact constraints and non-functional requirements.

**Phase 2: High-Level Design (The Happy Path)**

- The Candidate proposes the initial components and data flow.
- The Candidate provides the first Mermaid diagram.

**Phase 3: Deep Dive & API/DB Specifications**

- The Candidate defines the core API structure and Database Schema.
- The Interviewer zeroes in on a specific choice (e.g., "Why use a NoSQL DB here instead of a relational one?") and the Candidate defends the trade-offs.

**Phase 4: Scaling, Bottlenecks, and Failure Modes**

- The Interviewer introduces a massive scale constraint or a sudden component failure scenario.
- The Candidate adapts the design, explaining data partitioning, caching strategies, and async processing.
- The Candidate provides the final, scaled Mermaid architecture diagram.
