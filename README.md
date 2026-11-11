# Predicting the transcriptional response to an unseen gene knockdown

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/eersin1/virtual-cell/blob/main/virtual_cell.ipynb)

Silencing one gene in a human cell shifts thousands of others. Given a gene the model has
**never seen**, can its whole signature be predicted from a description of what the gene is?

Most drugs work by blocking one protein, which is roughly a knockdown — so predicting the
cell's response in software is the first step toward assessing a target before anyone runs the
experiment.

![One held-out gene: RPL23](figures/example_gene.png)

`RPL23` is a ribosomal protein, held out of training entirely. The model gets the **direction
right for all 25** of the genes that moved most: mitochondrial genes down, growth-arrest and
stress genes up — the textbook signature of blocking protein synthesis.

## Result

~2,057 silenced genes × 8,561 measured genes, 5-fold cross-validation **over genes**. Every
scored gene was absent from training, and each gene's own column is excluded from scoring.

| model | all genes | top 50 | knockdown-specific |
|---|---|---|---|
| average response (baseline) | 0.223 | 0.425 | 0.000 |
| protein sequence only (ESM-2) | 0.256 | 0.465 | 0.154 |
| annotations: ridge | 0.319 | 0.542 | 0.291 |
| annotations: nearest neighbours | 0.318 | 0.510 | 0.281 |
| annotations: neural network | 0.330 | 0.516 | 0.297 |
| **ensemble** | **0.345** | **0.555** | **0.313** |
| ensemble + ESM-2 | 0.345 | 0.553 | 0.314 |

**Read the last column.** Silencing almost any essential gene makes the cell unwell, and unwell
cells look alike — so "just predict the average response" is already a strong baseline at
0.223. The knockdown-specific metric removes that shared response from both prediction and
truth, measuring only what is distinctive about this gene. The baseline scores exactly 0 there
by construction.

![Model comparison](figures/models.png)

Two findings that cut against expectation, and are the more interesting half of the result:

- **The neural network barely beats ridge regression** (0.330 vs 0.319) and is *worse* on the
  genes that moved most (0.516 vs 0.542). This is consistent with
  [Ahlmann-Eltze et al., *Nature Methods* 2025](https://doi.org/10.1038/s41592-025-02772-6),
  who found deep models struggle to beat simple baselines at exactly this task.
- **ESM-2 adds nothing on top of curated annotations.** Sequence alone beats the baseline
  (0.256 vs 0.223), so a protein language model that has never seen an experiment does carry
  real information about what removing that protein does. But the ensemble does not improve
  when it is added (0.345 → 0.345). That information is already in what biologists wrote down.

![Where it works and where it fails](figures/breakdown.png)

The failures are not random. Accuracy tracks effect size: knockdowns that barely perturbed the
cell are mostly unpredictable, because there was little but noise to predict.

## Method

- **Data** — [Replogle et al., *Cell* 2022](https://doi.org/10.1016/j.cell.2022.05.013)
  genome-wide Perturb-seq, CRISPRi in K562, pseudobulk.
- **Gene descriptions** from three independent sources: Gene Ontology (TF-IDF → SVD, 256 dims),
  STRING v12 (symmetrically normalised adjacency → SVD, 256 dims), and ESM-2 650M mean
  residue embeddings (1,280 dims).
- **Models** predict the *residual* — the response minus the average training response — so
  capacity goes on what is distinctive rather than on the shared stress response.
- **Ensemble** of ridge / nearest-neighbours / MLP, weighted 0.4 / 0.2 / 0.4.

All scaling, PCA and feature standardisation are fit on training genes only.

## Limitations

- **One cell type.** K562 is a leukaemia line. Nothing here shows the model transfers.
- Pseudobulk averaging discards cell-to-cell variation.
- CRISPRi knocks a gene *down*, not out, and residual expression varies by gene.
- **GO and STRING are built from published experiments, so well-studied genes carry richer
  descriptions.** The model is partly rewarded for a gene being famous, and performance on a
  genuinely uncharacterised gene — the case you would actually care about — would be lower.
  Quantifying this is the best next experiment.
- Predictions are systematically too small in magnitude (ridge shrinkage; visible as a slope
  below 1 in the scatter above).
- Ensemble weights were chosen by hand.

## Prior work

Not methodologically novel. [GEARS](https://doi.org/10.1038/s41587-023-01905-6) and
[scGPT](https://doi.org/10.1038/s41592-024-02201-0) are the established models for this task;
this project does not compare against them directly. The contribution is a careful baseline
comparison and the negative result on protein-sequence features.

## Running it

Open in Colab with the badge above, or locally:

```bash
pip install numpy pandas scipy scikit-learn matplotlib h5py anndata torch fair-esm jupyter
jupyter notebook virtual_cell.ipynb
```

The notebook downloads its own data (~190 MB) and verifies each file parses to its end before
using it — a half-downloaded file is still a large file, and otherwise fails confusingly later.

The slow step is computing ESM-2 embeddings for ~1,900 proteins: a few minutes on a GPU,
considerably longer on CPU. It is cached to `cache/`, so it only happens once.

## Files

| | |
|---|---|
| `virtual_cell.ipynb` | the analysis |
| `figures/` | output figures |
