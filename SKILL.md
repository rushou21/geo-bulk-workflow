---
name: geo-bulk-workflow
description: Select public GEO bulk RNA-seq and microarray cohorts, analyze each dataset across relevant tissues and conditions, and compare results across cohorts. Use for differential expression, enrichment, coexpression networks, candidate gene selection, or prediction model validation.
---

# Bioinformatics Bulk Transcriptomics

Read the relevant section of the [workflow and scoring rubric](references/workflow.txt) for exact inputs, outputs, and decision rules. For a full project, follow the dependencies below. For a single task, reuse verified outputs from earlier steps.

Before running a stage, check its [environment and R package requirements](README.md#prepare-the-environment). The [installation instructions](README.md#install-and-invoke-the-skill) also list optional companion skills.

## Define the comparison and select cohorts

Describe the species, tissue, sampling time, target group, reference group, and independent biological unit in one sentence. Assign a `contrast_id` and state what a positive effect means. Define each comparison explicitly, including paired treatment samples and comparisons between responders and nonresponders.

Before scoring a comparison, verify that both expression data and per-sample group assignments can be read or downloaded publicly without logging in, applying for access, or contacting the authors. Open the file or query the public endpoint and check the file type, sample identifiers, and group fields. For large files, inspect the header or archive manifest. In the Excel shortlist, record the expression and metadata links, filenames or retrieval method, and verification date. One link is sufficient if a file contains both data types. Prefer expression matrices or per-sample quantification files. Label public FASTQ or CEL files that still require processing as "raw data only; preprocessing required." Exclude records with only a paper or study summary and records with broken or unverified download links from the top five.

Rank up to five eligible comparisons using the six criteria in the rubric. Prioritize human cohorts and consider mouse cohorts when appropriate to the study question. Record evidence for each score and flag unverified items. Assign discovery, validation, or backup roles before inspecting differential expression or predictive performance. For a validation cohort, verify tissue, outcome definition, measurement scale, and absence of participant overlap with the discovery cohort. Count independent biological units or complete pairs.

## Establish coverage for each selected dataset

For a full analysis, inventory the tissues or anatomical regions, groups, time points, and biological units in each selected cohort. Plan and run every estimable comparison relevant to the user's question, including regions or time points available in only one cohort. The top-five limit applies to shortlisting. Record each comparison, its status, sample counts, and any reason for omission (workflow section 1.5). Run the requested stages in order. If a stage cannot run, give the specific missing input or failed operation and continue the remaining stages. Respect an explicitly narrower request.

## Prepare inputs

- Use the project directories `raw_data`, `clean_data`, `scripts`, `results`, and `report`. Follow the file conventions in the workflow. The prefix `P` stands for `<GSE>_<contrast_id>`.
- Use `gene_id` to identify expression rows and `sample_id` to identify samples. Retain participant IDs for repeated samples. Establish an unambiguous match using GEO sample accessions (GSM) or the authors' mapping, then explicitly align matrix columns and metadata rows. List ambiguous matches for review.
- Record the measurement units, platform, and processing history. For RNA-seq count models, use unnormalized counts or properly imported quantification estimates. Prepare a separate transformed expression matrix for quality control and WGCNA. For microarrays, check platform annotation, normalization, and log2 transformation, and retain the probe-to-gene mapping. Store stable gene IDs alongside display symbols.

## Follow the analysis dependencies

1. **Differential expression and enrichment.** Fit the specified contrast, accounting for pairing and estimable confounders. Save the full results and the genes passing the prespecified primary rule. Select the fold-change cutoff for the study question. It can be relaxed when smaller changes are relevant. Report how many genes pass the false discovery rate (FDR) criterion alone and how many also pass any chosen fold-change cutoff. For GO and KEGG overrepresentation analysis, use the significant genes and a background of eligible tested genes that map to the relevant database. For GSEA, use all tested genes with valid ranking scores and interpret the normalized enrichment score (NES) against the specified contrast direction.
2. **Coexpression networks.** Run WGCNA on the transformed expression matrix independently of differential expression results. Skip network analysis below 15 independent samples and prefer at least 20. Select variable genes using a prespecified rule such as median absolute deviation (MAD). Choose the soft threshold using model fit and network connectivity, and document how an unmet target was handled. Assess positive and negative module associations using a prespecified multiple-testing correction.
3. **Candidate genes and STRING.** Intersect significant differentially expressed genes with the genes belonging to eligible modules. Use this list for STRING. The prediction branch uses the feature rule in workflow section 10. Configure STRING for the study species. A confidence threshold of 0.4 corresponds to `score_threshold = 400` in STRINGdb. Record unmapped genes, isolated nodes, and networks with no edges.
4. **Empty results.** An empty significant-gene list still permits GSEA and, when eligible, WGCNA. If a module or candidate list is unavailable, record the reason. Continue classification with training-filtered expression features as described in section 10. The intersection of features selected by LASSO, random forest, and SVM-RFE is a separate list used by the nomogram. An empty three-method intersection ends that model branch.

## Relate results across datasets

Once the within-cohort comparisons are accounted for, align comparable contrasts and interpret the relationship using workflow section 12. Report both agreement and disagreement in gene effects and pathways, alongside results unique to individual datasets. When exposures or outcomes differ, describe shared biology as a hypothesis. For an independent validation cohort, freeze the discovery result before testing it there.

## Run machine learning for labeled comparisons

For a full analysis, run LASSO, random forest, and SVM-RFE for each planned target-versus-reference comparison with observed group labels and usable expression data. Use the actual group labels and report what the classifier predicts. Evaluate internally when an independent validation cohort is unavailable. A cohort reserved for external validation receives the frozen model. Honor a narrower user request.

- Prespecify the outcome, prediction time, feature selection rules, and evaluation design. Resample by participant or pair and choose a fold count compatible with the smaller class.
- To estimate performance internally, repeat all data-dependent modeling steps within each outer training set or bootstrap sample. Learn transformations, select features, and tune within training data. If the feature rule uses differential expression and modules, recompute their intersection in the training data and apply the same fallback rule in each split. Keep records of repeats that fail to produce a model. A five-fold tuning score for the final model does not evaluate the full pipeline.
- For external validation, features can be selected and the final model fitted on the entire development cohort. Freeze the features, preprocessing rules, coefficients, and decision threshold before applying them to the independent cohort. A list selected with all development participants cannot be reused to estimate unbiased internal performance on those same participants.
- Case/control labels support binary classification. Survival modeling additionally requires an event definition, time, event status, and a suitable survival model. For case-control sampling, specify the target population prevalence and account for sampling when interpreting absolute risk, calibration, or decision net benefit. Identify exploratory candidates, internal performance estimates, and independent validation results separately in the report.

Write the report in English. Include actual sample sizes, contrasts, input sources, parameters, skipped or incomplete steps, and observed results. Save reproducible R scripts with English comments, random seeds, and software versions.
