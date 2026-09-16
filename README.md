# 🔎 PhenoPrompt

> **Prompt-based clinical phenotype discovery.** Build a phenotype space from clinical notes,
> then query it in natural language — type a clinical concept and retrieve the patient clusters
> where it is enriched, with no disease-specific algorithm required.

[![Live demo](https://img.shields.io/badge/demo-streamlit-FF4B4B)](https://phenoprompt-skausar-aivancity.streamlit.app/) [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Shafiya0101/phenoprompt) [![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://github.com/Shafiya0101/phenoprompt/blob/main/LICENSE)

> Extends an earlier course project on structured-data phenotyping, [mimic-phenotyping](https://github.com/Shafiya0101/mimic-phenotyping), from a structured ICU
> table to unstructured clinical text — and adds the prompt-based query layer.

---

## What this is

A three-stage pipeline that turns raw clinical notes into a *queryable* map of patient
phenotypes:

```
Stage 1  medkit NER          notes ──▶ clinical entities (disorders, meds, findings)
Stage 2  embed + cluster     entity profiles ──▶ UMAP map + HDBSCAN phenotype clusters
Stage 3  prompt query (RAG)  free-text prompt ──▶ ranked, interpretable phenotype reports
```

The contribution is **Stage 3**: an entity-augmented retrieval layer (inspired by CLEAR,
López et al., *npj Digital Medicine* 2025) that scores each cluster by how **enriched** the
query's clinical entities are within it — prevalence × lift, so common entities can't pull in
unrelated clusters — then synthesises a readable phenotype report. Any condition is queryable
from a single index built **without** disease-specific specification, in contrast to the
"one algorithm per disease" norm in LLM phenotyping.

## Repository layout

```
notebooks/
  stage1_medkit_ner.ipynb            entity extraction (Colab + Google Drive)
  stage2_embedding_clustering.ipynb  LSA embedding, UMAP, HDBSCAN, phenotype mixes
  stage3_prompt_rag.ipynb            prompt-based query interface
phenoprompt_app.py                   Streamlit query app
data/phenoprompt/
  stage1_outputs/   commit: entity_count_matrix.csv, entity_mentions.csv, notes.csv
  stage2_outputs/   commit: phenotype_profiles.json, cluster_assignments.csv, umap_2d_coords.csv
requirements.txt                     app dependencies
```

## Running the notebooks (Colab)

The notebooks read and write a shared Google Drive folder so results survive session
disconnects. Run them **in order**:

1. `stage1_medkit_ner.ipynb` — downloads the corpus, runs medkit, writes
`MyDrive/phenoprompt/stage1_outputs/`. (Slowest stage.)
2. `stage2_embedding_clustering.ipynb` — clusters and writes
`MyDrive/phenoprompt/stage2_outputs/` (including `phenotype_profiles.json`).
3. `stage3_prompt_rag.ipynb` — loads the above and answers prompts. Defaults to a no-API
"fallback" report writer; set `LLM_BACKEND` to `anthropic`/`openai` for LLM-written
cluster labels and narratives.

## Running the app

```
pip install -r requirements.txt
streamlit run phenoprompt_app.py
```

The app loads the committed files under `data/phenoprompt/`. Download the six output files
from your Drive into those folders (see the `README.txt` in each) before deploying to
Streamlit Cloud, which serves from the GitHub repo rather than from Drive.

A live instance is hosted at **[phenoprompt-skausar-aivancity.streamlit.app](https://phenoprompt-skausar-aivancity.streamlit.app/)**.

## Data

Runs on the **synthetic** `AGBonnet/augmented-clinical-notes` corpus (HuggingFace) — no
PhysioNet credentialing or PHI restriction. The full corpus is 30,000 synthetic notes;
28,562 (95.2%) contain at least one affirmed clinical entity and are carried forward to
clustering and retrieval. Porting to real MIMIC-III/IV discharge summaries is planned and
would require credentialed access.

## Status & known limitations

The pipeline runs end to end and has been scaled from the original 500-note pilot to the
full 28,562-note corpus, with entity normalisation and a UMLS-grounded NER variant added
since the initial preliminary results.

| Configuration | Clusters | Noise | Silhouette | DBCV |
|---|---|---|---|---|
| 500-note pilot | 6 | 69.6% | ≈0.27 | ≈0.04 |
| Full corpus (raw entity forms) | 76 | 76.2% | 0.66 | 0.08 |
| **Full corpus (canonical concepts, current)** | **113** | **69.1%** | **0.669** | **0.123** |

Key findings from scaling up:
- **Entity fragmentation, not corpus size, was the binding constraint.** Scaling from 500 to
  30,000 notes on raw entity surface forms improved cohesion (silhouette 0.27 → 0.66) but made
  coverage *worse* (noise 69.6% → 76.2%). Collapsing 165 raw surface forms onto 78 canonical
  concepts (by keying each mention on its rule identifier, no hand-written synonym table
  required at this stage) is what recovered coverage (→69.1% noise) and raised DBCV by ~50%.
- **UMLS-grounded NER (scispaCy + UMLS linker)** was piloted as an alternative to the
  rule-based medkit extraction: 6,963 linked concepts vs. 78 rule-based concepts — much wider
  coverage, though the most frequent linked terms mix in generic/administrative labels
  alongside genuine clinical ones, so this hasn't replaced the rule-based path yet.
- **Assertion propagation** was fixed to run negation/hypothesis detection on syntagmas
  (clauses) rather than individual entities, correctly activating negation filtering at scale
  (97.0% affirmed, 2.8% negated, 0.2% hypothetical).
- A **granularity sweep** over `min_cluster_size ∈ {5,10,20,30,50}` found silhouette stays flat
  (0.61–0.69) across the range, so it can't alone pick a configuration; `min_cluster_size=30` is
  used as it balances 113 interpretable clusters against coverage. Merging clusters via
  `cluster_selection_epsilon` (tested at 0.5) was a negative result — it collapsed 19,766 of
  28,562 notes into one undifferentiated cluster while silhouette fell to 0.0068.

**Note on the query-time synonym layer:** the retrieval app (`rag_app.py`) additionally uses a
small, hand-written synonym dictionary (`SYNONYMS`) to map colloquial or abbreviated query
wording — e.g. "sob"/"breathless" → "shortness of breath", "renal"/"kidney" → "chronic kidney
disease" — onto canonical entities at *query time*. This is separate from the rule-ID-based
canonical-concept normalisation used when *building the index* (described above), which needs
no synonym table. In an ablation on the 500-note pilot, 2 of 4 colloquial test queries
("breathless", "fluid overload") returned zero results under exact matching; the query-time
synonym layer recovered all four to their full top-5 result set.

**Remaining gaps:**
- No comparison yet to established baselines (PheKB, PheNorm, Phe2vec).
- Neither extraction path (rule-based medkit or the UMLS/scispaCy variant) has gold-standard
  NER precision/recall; a pilot expert-annotation round is under way.
- ≈69% of notes remain unclustered (noise); clinical validation on credentialed, real EHR data
  with expert review is the primary next step.
- Agentic query-reformulation accuracy is not yet quantified; LLM-judged phenotype coherence
  (500-note pilot: 5 of 6 clusters rated a recognisable clinical phenotype, mean precision@10 =
  0.53) uses the same model family as generation (a self-evaluation loop) and has not yet been
  recomputed for the 113-cluster full-corpus run.

## License

MIT — see [LICENSE](https://github.com/Shafiya0101/phenoprompt/blob/main/LICENSE).
