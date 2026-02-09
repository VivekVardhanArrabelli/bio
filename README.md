BioOracle — Build Spec & Execution Plan
The Problem
There are 36+ million papers on PubMed. No human — or team of humans — can read even a fraction. Connections between fields go unnoticed for years. A cardiologist doesn't read oncology papers. A biochemist doesn't read clinical trial results.
Don Swanson proved in the 1980s that you could discover real treatments (fish oil → Raynaud's disease) just by connecting papers nobody had read together. He was right. It worked. But he did it by hand.
Existing tools (Arrowsmith, BITOLA, LitLinker) use keyword co-occurrence from the 2000s. BenevolentAI does this with AI but charges millions and is proprietary. No open-source tool uses LLM reasoning to judge whether a transitive biological chain actually makes mechanistic sense.
BioOracle fills that gap. Open source. Free. Available to every researcher on earth.

What It Does
Input: The entire PubMed abstract corpus (focus areas: oncology, neurodegeneration, metabolic disease, autoimmune, cardiovascular, rare diseases)
Output: A ranked database of novel drug repurposing hypotheses with full evidence chains and citations.
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
│                      DATA INGESTION                         │
│                                                             │
│  PubMed API ──→ Abstract Store (SQLite/Parquet)             │
│  ClinicalTrials.gov API ──→ Trial Store                     │
│  DrugBank (open subset) ──→ Drug Metadata                   │
│  UniProt/KEGG (open) ──→ Pathway Data                       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  EXTRACTION LAYER                           │
│                                                             │
│  Claude Agents read abstracts in parallel                   │
│  Extract structured triples:                                │
│    (Drug X) ──[inhibits]──→ (Protein Y)                     │
│    (Protein Y) ──[upregulated_in]──→ (Disease Z)            │
│    (Gene A) ──[regulates]──→ (Pathway B)                    │
│    (Drug X) ──[treats]──→ (Disease W)                       │
│                                                             │
│  Output: JSONL files of structured relations                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  KNOWLEDGE GRAPH                            │
│                                                             │
│  Nodes: Drugs, Proteins, Genes, Pathways, Diseases          │
│  Edges: Extracted relations with PMID provenance            │
│  Store: SQLite + NetworkX (or Neo4j if time permits)        │
│  Size target: ~5-10M edges from ~2-3M abstracts             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                HYPOTHESIS ENGINE                            │
│                                                             │
│  1. Transitive path finding:                                │
│     Drug → Protein → Disease (no direct Drug→Disease link)  │
│     Drug → Gene → Pathway → Disease                         │
│                                                             │
│  2. LLM Reasoning Filter (THE KEY DIFFERENTIATOR):          │
│     Claude evaluates each candidate chain:                  │
│     - Does this mechanism make biological sense?            │
│     - Are there confounding factors?                        │
│     - How strong is each link in the chain?                 │
│     - What's the confidence level?                          │
│                                                             │
│  3. Novelty Check:                                          │
│     - Is Drug X already known to treat Disease Z?           │
│     - Are there existing clinical trials?                   │
│     - Has this specific chain been proposed before?         │
│                                                             │
│  4. Scoring & Ranking:                                      │
│     - Biological plausibility (LLM score)                   │
│     - Evidence strength (# supporting papers, journal IF)   │
│     - Novelty (inverse of existing connections)             │
│     - Drug accessibility (FDA approved > Phase III > etc)   │
│     - Chain length penalty (shorter = more plausible)       │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  VALIDATION MODULE                          │
│                                                             │
│  Retrospective validation:                                  │
│  - Feed system papers BEFORE a known discovery              │
│  - Can it "rediscover" known repurposing successes?         │
│    • Thalidomide → multiple myeloma                         │
│    • Metformin → cancer prevention                          │
│    • Sildenafil → pulmonary hypertension                    │
│    • Aspirin → colorectal cancer prevention                 │
│    • Baricitinib → COVID-19 (BenevolentAI's discovery)      │
│  - Report: "System ranked known discovery at position X     │
│    out of Y candidates"                                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    WEB INTERFACE                            │
│                                                             │
│  Next.js app:                                               │
│  - Browse ranked hypotheses by disease area                 │
│  - Search: "Show me hypotheses for Parkinson's"             │
│  - Filter by confidence, novelty, drug status               │
│  - Full evidence chain with clickable PubMed links          │
│  - Interactive knowledge graph visualization                │
│  - Export: CSV, JSON, PDF reports                           │
│  - API endpoint for programmatic access                     │
└─────────────────────────────────────────────────────────────┘
Tech Stack
ComponentTechnologyWhyData ingestionPython + PubMed E-utilities APIStandard, free, well-documentedAbstract storageParquet files + SQLiteFast columnar reads, no server neededRelation extractionClaude API (batch mode)The core reasoning capabilityKnowledge graphSQLite + NetworkXSimple, portable, no infra neededPath findingNetworkX + custom algorithmsWell-tested graph algorithmsHypothesis scoringClaude APIBiological plausibility judgmentWeb frontendNext.js + TailwindFast to build, good DXGraph visualizationD3.js or Cytoscape.jsInteractive knowledge graphDeploymentDocker + GitHub Pages/VercelEasy self-hostingAPIFastAPISimple REST endpoints
Data Sources (All Free/Open)
SourceWhatAccessPubMed/MEDLINE36M+ abstractsE-utilities API (free, rate-limited)PubMed FTP bulkAnnual baseline + daily updatesftp.ncbi.nlm.nih.govClinicalTrials.govActive/completed trialsAPI (free)DrugBank (open)Drug targets, mechanismsCC-BY-NC downloadUniProtProtein functions, interactionsFree APIKEGG (limited)PathwaysFree academic APIMeSHMedical ontologyFree downloadChEMBLBioactivity dataFree download

Agent Allocation (20-25 Parallel Claude Code Agents)
Cluster 1: Data Pipeline (4 agents, Days 1-3)
Agent 1.1 — PubMed Ingester

Downloads PubMed baseline XML files (bulk FTP)
Parses XML → structured records (PMID, title, abstract, MeSH terms, year, journal)
Stores in Parquet files partitioned by year
Target: 2-3M abstracts in focus disease areas
Filters by MeSH terms: neoplasms, neurodegenerative, metabolic, cardiovascular, autoimmune

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
This is crucial for the "has this already been tried?" check

Cluster 2: Relation Extraction (8 agents, Days 2-5)
This is the biggest workload. Each agent owns a disease domain and processes ~300-400K abstracts.
Agent 2.1 — Oncology abstracts
Agent 2.2 — Neurodegeneration (Alzheimer's, Parkinson's, ALS, etc.)
Agent 2.3 — Metabolic disease (diabetes, obesity, NAFLD)
Agent 2.4 — Cardiovascular disease
Agent 2.5 — Autoimmune / inflammatory
Agent 2.6 — Rare diseases
Agent 2.7 — Infectious disease
Agent 2.8 — Overflow / reprocessing / quality control
Each agent's workflow:
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
Score novelty: NOVEL (no prior mention), EMERGING (< 3 papers), KNOWN (> 3 papers)


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

GET /hypotheses?disease=parkinsons&min_score=0.7
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
TimeAgentsTaskMorning1.1, 1.2, 1.3, 1.4Start all data downloads (PubMed bulk, DrugBank, UniProt, trials)Morning6.3Set up repo, CI/CD, Docker scaffoldingMorning7.1Set up monitoring, logging infrastructureAfternoon1.1Parse first batch of PubMed XMLs → ParquetAfternoon1.2DrugBank parsing complete, start ChEMBLEvening2.1-2.8Begin extraction on first available abstracts
Day 1 milestone: Data pipeline running, first 100K abstracts ingested, extraction started.
Day 2 (Tuesday): Extraction Sprint
TimeAgentsTaskAll day2.1-2.8Full extraction sprint (~50K abstracts each)All day1.1Continue PubMed ingestionMorning1.3Entity normalization mappings completeAfternoon3.1Begin building graph from first extraction outputsEvening5.2Start quality audit on early extractions
Day 2 milestone: 500K+ abstracts processed, graph construction started.
Day 3 (Wednesday): Graph & First Hypotheses
TimeAgentsTaskAll day2.1-2.8Continue extraction (targeting 1.5M cumulative)Morning3.1, 3.2Graph construction + indexingAfternoon4.1First path-finding run on partial graphAfternoon6.1Start backend API developmentEvening4.2First LLM reasoning evaluation on candidate paths
Day 3 milestone: First draft hypotheses generated from partial data. Graph has 1M+ edges.
Day 4 (Thursday): Hypothesis Engine Full Run
TimeAgentsTaskMorning2.1-2.8Extraction winding down (~2M cumulative)All day4.1-4.4Full hypothesis pipeline: find → reason → check → scoreMorning6.1Backend API functionalAfternoon6.2Frontend development beginsEvening5.1Start retrospective validation
Day 4 milestone: Ranked hypothesis list exists. API serves results. Frontend scaffold up.
Day 5 (Friday): Integration & Validation
TimeAgentsTaskMorning4.1-4.4Re-run hypothesis engine on complete graphAll day5.1Complete retrospective validation (all 5 test cases)All day6.2Frontend: browse, search, detail pagesAfternoon6.1Graph visualization endpointEvening3.2Final graph statistics
Day 5 milestone: Full hypothesis database. Validation results. Working frontend.
Day 6 (Saturday): Polish & Edge Cases
TimeAgentsTaskMorning6.2Frontend polish: graph viz, filters, mobile responsiveMorning4.4Generate detailed write-ups for top 500 hypothesesAfternoon5.2Final quality auditAfternoon7.2Set up daily update pipelineEvening6.3Documentation, deployment scriptsEveningALLIntegration testing
Day 6 milestone: Production-ready. Documented. Self-updating.
Day 7 (Sunday): Launch
TimeAgentsTaskMorning6.3Final deployment, SSL, domain setupMorning6.2Last UI fixesAfternoon6.3Write launch blog post / HN submissionAfternoonALLEnd-to-end smoke testsEvening-LAUNCH

Key Risks & Mitigations
RiskImpactMitigationPubMed rate limitingSlow data ingestionUse bulk FTP download, not API for initial loadClaude API costs for 2M+ abstractsBudget overrunBatch API (50% cheaper), extract only from abstracts not full textExtraction quality is poorBad hypothesesQuality audit loop (Agent 5.2), iterative prompt improvementToo many false positive hypothesesNoise overwhelms signalAggressive LLM reasoning filter + high threshold for plausibilityEntity normalization failuresFragmented graphUse established ontologies (MeSH, UniProt IDs), fuzzy matchingGraph too sparse in some areasMissing connectionsSupplement with DrugBank known interactions as seed edgesRetrospective validation failsNo credibilityTune scoring weights until validation passes, be transparent about limitationsFrontend not ready in timeNo usabilityMinimum viable: static JSON files + simple search page

Success Metrics
Must-have (for hackathon demo)

 1M+ abstracts processed
 500K+ extracted relations in knowledge graph
 1000+ ranked novel hypotheses
 At least 2/5 retrospective validations succeed (system ranks known discovery in top 100)
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

 A researcher sees a hypothesis and says "that's interesting, I want to test that"
 One hypothesis leads to an actual experiment or clinical trial
 System rediscovers a known drug repurposing that took humans years to find


Cost Estimate (Claude API)
TaskAbstracts/QueriesEst. TokensEst. CostRelation extraction2M abstracts~4B tokens~$4,000 (batch)Hypothesis reasoning50K candidates~500M tokens~$500Scoring & write-ups5K hypotheses~100M tokens~$100Validation runs5 scenarios~50M tokens~$50Total~$4,650
Well within a $100K credit budget. Leaves room for iteration and re-runs.

Why This Wins the Hackathon

Ambitious but scoped: Drug repurposing is a real, important problem. We're not solving all of biology — we're solving one specific thing well.
Perfect for agent swarm: 20+ agents working in parallel on genuinely independent tasks. Not a toy demo of parallelism.
Demonstrates LLM advantage: The reasoning filter is something no keyword/statistical tool can do. This is Claude doing what Claude is uniquely good at.
Immediately useful: Researchers can browse hypotheses the day it launches. No training, no setup, just search.
Validatable: Retrospective validation proves the system works (or exposes honestly where it doesn't).
Open source impact: Democratizes what a $400M company keeps proprietary. Every underfunded lab in every country gets access.
Great demo: "Our system would have discovered baricitinib for COVID-19 — 2 years before BenevolentAI did" is a killer demo line.
