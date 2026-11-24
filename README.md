# Bioinformatics Bulk Transcriptomics

`geo-bulk-workflow` is a Codex skill for selecting public GEO bulk RNA-seq or expression microarray cohorts and analyzing a defined biological comparison. Use it for a single task, such as ranking datasets, or for a study that proceeds from differential expression to independent model validation. The repository supplies instructions and analysis templates. Codex writes and runs study-specific R scripts when an analysis is requested.

## Install and invoke the skill

Run this command at the root of the research project.

```sh
git clone https://github.com/rushou21/geo-bulk-workflow.git .agents/skills/geo-bulk-workflow
```

Alternatively, extract the repository ZIP into the project so that the skill file is at `.agents/skills/geo-bulk-workflow/SKILL.md`. Open a new Codex chat in that project and invoke `$geo-bulk-workflow`. The skill is self-contained. A separate bioSkills installation, additional Codex skills, and a skill installer are optional.

If your agent already has bioSkills installed, it can use these companion skills for specific tasks. The names below are skill identifiers. Install R packages separately as described under [Prepare the environment](#prepare-the-environment).

| Task | Optional companion skill |
| --- | --- |
| Align matrix columns with sample metadata and set up the contrast | `bio-expression-matrix-metadata-joins` |
| Fit a differential expression model for RNA-seq counts | `bio-differential-expression-deseq2-basics` |
| Draw and annotate a clustered expression heatmap | `bio-data-visualization-heatmaps-clustering` |

**Example request**

> Use $geo-bulk-workflow to find human GEO cohorts comparing synovial tissue from rheumatoid arthritis cases and controls. Rank up to five eligible comparisons using the scoring rubric. For each comparison, give the independent sample counts, expression and sample-group links, verification date, and score. Stop after cohort selection.

Before ranking a comparison, the skill checks that both expression data and per-sample group assignments are publicly retrievable. It prioritizes processed expression data, records when only raw files are available, and considers mouse cohorts when suitable human data are unavailable.

## Prepare the environment

| Task | What you need |
| --- | --- |
| Install the skill and review a workflow | A Codex installation that can load project skills and read the project directory. Git is needed only for the clone command. |
| Search GEO and verify files | Internet access to GEO and its linked public file hosts. A browser or HTTP client suffices for manual checks. To automate retrieval or write the Excel shortlist in R, use the packages below. |
| Run an analysis | R with `Rscript` available on the command line, a writable R library and project directory, and disk space for the chosen expression files and outputs. Choose an R release supported by the Bioconductor release and the packages you need. |
| Install packages or access online resources | Internet access to CRAN and Bioconductor for installation. Specific stages also retrieve KEGG pathways, MSigDB gene sets via `msigdbr`, or STRING interactions. If packages must be compiled from source, use Rtools matching R on Windows, Xcode Command Line Tools on macOS, or the needed compiler and development libraries on Linux. Some source packages also require a Fortran compiler. |

Check the R command from a terminal before installing packages.

```sh
Rscript --version
```

If your terminal cannot find `Rscript`, add the R installation's `bin` directory to your `PATH`, or call `Rscript` by its full path.

The following packages cover the workflow. Each row applies only to the stage and input type you choose. Platform-specific probe annotation or raw-file processing may require additional packages identified from the GEO platform record.

| Stage | CRAN packages | Bioconductor packages |
| --- | --- | --- |
| Retrieve GEO records and write an Excel shortlist | `openxlsx` for Excel output | `GEOquery` for GEO retrieval |
| Annotate genes and probes | | `AnnotationDbi` with `org.Hs.eg.db` (human) or `org.Mm.eg.db` (mouse). For arrays, use the matching platform annotation when needed. |
| Prepare and test expression data | | `DESeq2` or `edgeR` for counts, `limma` for normalized log-scale microarray data, or `edgeR` + `limma` for voom. Use `tximport` for supported transcript quantifications, or `affy` / `oligo` for supported raw arrays. |
| GO and KEGG enrichment, Hallmark GSEA | `msigdbr` for species-appropriate MSigDB gene sets | `clusterProfiler`, `enrichplot`, and the relevant organism annotation |
| WGCNA and candidate intersections | `WGCNA`. Base R provides `intersect()` for gene lists. | |
| STRING network | `igraph` for network handling and plots | `STRINGdb` |
| Feature selection | `glmnet` for LASSO, `ranger` for random forests, `caret` for resampling and recursive feature elimination, `e1071` for a linear SVM with a corresponding coefficient-based ranking function | |
| Nomogram and validation | `rms` for fitting and plotting, `pROC` for binary ROC-AUC, `dcurves` for decision curves, and `survival` for time-to-event analyses | |
| Additional figures | `ggplot2`, `pheatmap`, and `ggVennDiagram` for expression plots, heatmaps, and intersections | |

Install the following from an **R console** for GEO retrieval and Excel output.

```r
install.packages("BiocManager", repos = "https://cloud.r-project.org")
BiocManager::install(c("GEOquery", "openxlsx"), ask = FALSE, update = FALSE)
```

For a human microarray cohort analyzed with limma, GO/KEGG, and Hallmark GSEA, add these packages.

```r
BiocManager::install(
  c("limma", "AnnotationDbi", "org.Hs.eg.db", "clusterProfiler", "enrichplot",
    "msigdbr", "ggplot2", "pheatmap"),
  ask = FALSE, update = FALSE
)
```

Use `BiocManager::install()` for further packages from either column. This also resolves dependencies across CRAN and Bioconductor, including those of `WGCNA`. Check release compatibility with `BiocManager::valid()`. Record `sessionInfo()` with each analysis, along with the dataset, platform annotation, gene-set and database versions. [Bioconductor installation guide](https://bioconductor.org/install/) and [CRAN downloads](https://cran.r-project.org/).

If the study proceeds through coexpression analysis, STRING, binary feature selection and a nomogram, add the packages for those stages.

```r
BiocManager::install(
  c("WGCNA", "ggVennDiagram", "STRINGdb", "igraph", "glmnet", "ranger",
    "caret", "e1071", "rms", "pROC", "dcurves"),
  ask = FALSE, update = FALSE
)
```

To verify that the same non-interactive R process used by the agent can load the packages, run this in the project terminal. Add any other packages your analysis will use to the list.

```sh
Rscript --vanilla -e "p <- c('GEOquery', 'openxlsx'); ok <- vapply(p, requireNamespace, logical(1), quietly = TRUE); print(ok); stopifnot(all(ok))"
```

## Workflow and outputs

Define the contrast and biological replicate before scoring cohorts. For a full study, complete the relevant tissue, condition, and time comparisons within each selected dataset, then compare compatible results across datasets. The workflow covers dataset selection, input preparation and quality control, differential expression, enrichment, WGCNA, gene-list intersection, STRING visualization, feature selection, and prediction. Each stage specifies its inputs, expected files, and conditions for proceeding. Full analyses include machine learning on observed group labels, with internal evaluation when no external cohort is available. An empty DEG–module intersection switches the input to expression features selected within each training split. For prediction, fit all data-dependent steps within the training portion of each resample. Independent validation uses the frozen model.

The analysis project holds data in `raw_data/` and `clean_data/`, generated R scripts in `scripts/`, analysis outputs in `results/`, and the English report in `report/`. See the [skill instructions](SKILL.md) for dependencies between stages and the [full workflow](references/workflow.txt) for the cohort scoring rubric, prompts, and output filenames.

## License

Released under the [MIT License](LICENSE).
