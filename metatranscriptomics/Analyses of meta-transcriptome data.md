# Analyses of meta-transcriptome data

## Differential expression analyses
After obtaining the table of counts and mapping statistics, a sensible next step is to perform differential expression analyses on your data. This is commonly done in R with the aid of packages such as:
 - [edgeR](https://bioconductor.org/packages/release/bioc/html/edgeR.html) ([Robinson, McCarthy & Smyth, 2010](https://doi.org/10.1093/bioinformatics/btp616))
 - [DESeq2](http://www.bioconductor.org/packages/release/bioc/html/DESeq2.html) ([Love, Huber & Anders, 2014](https://doi.org/10.1186/s13059-014-0550-8))
 - [limma](https://bioinf.wehi.edu.au/limma/) ([Ritchie et al., 2015](https://doi.org/10.1093/nar/gkv007))

In this document, we will briefly go through all 3 methods. Depending on your study design, you may be restricted in which method you might use. Anecdotally, I find **edgeR** implementation to be more flexible. 

---

### Preparation
Before diving into any of the analyses, it is important to set up the count matrix, related sample-wise summaries (sample metadata and library sizes), study design, and gene-wise features. In this case, features would mean annotations, or gene-wise statistics (e.g. length, strand, GC content, etc). The code below assumes several things:
 - obtained a table of counts and some gene-wise features from `featureCounts`
 - obtained sample-wise summary statistics based on code in the previous Markdown document
 - have a simple multi-group study design (more complex study designs are discussed in brief)
 - have a basic understanding of how to use R

```R
# Set up working environment
library(edgeR)
library(DESeq2)
library(limma)
library(tidyverse) # This is mainly for convenience functions

# Read relevant objects into your working environment
fc_data <- read_tsv("gene_counts.txt") # Table from featureCounts
lib_size <- read_tsv("lib_size.txt") # Library size for total number of reads per sample
metadata <- read_tsv("sample_metadata.txt") # Optional, but often helpful

# Set up count matrix
# Essentially, we are only retaining the Gene ID (or Node ID equivalent from prodigal) and gene counts per sample.
count_data <- fc_data %>%
  select(-c("Chr", "Start", "End", "Strand", "Length")) %>%
  column_to_rownames("Geneid")

# Set up gene features
gene_features <- fc_data %>%
  select(c("Geneid", "Chr", "Start", "End", "Strand", "Length")) %>%
  # left_join(gene_annotations, by = c("Geneid" = "node")) %>%
  column_to_rownames("Geneid")
# Note: You can also include annotation data here if you want to (as commented out above). This assumes your "Geneid" are inherited from the "node" from prodigal outputs. 

# Set up sample library size information
lib_size <- pull(lib_size, var = total_reads, name = sample)
# Note: Do make sure to check if your sample arrangement and names are the same throughout the process (i.e. sample as column names in "count_data" are identical to sample names in library size). Modify as appropriate for consistency. None of the workflows presented here make any guesses regarding gene and sample IDs and simply subset/match according to the row index. This is important as you do not want to lose relevant annotation/feature information based on incorrect subsets down the workflow when it comes to compiling the results.

```
> *A word of caution*
> 
> Make sure to check that the arrangement of your gene and sample IDs are **CONSISTENT** throughout the process (i.e. sample as column names in "count_data" are **IDENTICAL** to sample names in library size and metadata). Modify as appropriate for consistency. None of the workflows presented here make any guesses regarding gene and sample IDs and simply subset/match according to the row index. This is important as you do not want to lose relevant annotation/feature information based on incorrect subsets down the workflow when it comes to compiling the results. You can try to avoid the issue by doing of the following:
> - Sanity checks using the `summary()`, `nrow()`, `ncol()` functions to ensure you have equivalent number of genes and samples across relevant data.
> - Impose a strict arrangement of gene and sample IDs, such as using any of the data as a reference for sample IDs (from lib_size, or metadata) and sorting gene IDs.

---

### edgeR
`edgeR` stores all of its information in a special type of list: DGEList. To start, we will need to populate a DGEList with a matrix of counts, gene features, and library size.

1. Create DGEList object and inspect samples

```R
# Create DGEList
dge <- DGEList(
  counts = count_data, 
  lib.size = lib_size, 
  samples = metadata,
  genes = gene_features
)

# Visualize ordination between samples
plotMDS(dge)
```
> *A word on library sizes*
>
> We specify library size here as the total number of reads from a sample. By default, if we did not specify a vector for library size, `edgeR` will assume the library size is equal to the sum of counts in each column. This is not an unreasonable estimate if you have very well mapped reads. However, in the case of environmental meta-transcriptomics, it is often the case that there are substantially more unmapped than mapped reads. This is especially so if you are restricting your mapping reference to a particular lineage (e.g. mapping to prokaryotic MAGs only). In this case, the column sums of the count matrix is a substantial underestimation of the meta-transcriptome. As such, we opt to use the total number of reads instead of number of mapped reads.
>
> For this reason, when we filter the count data below, we opt to retain the library size. If not, it will be recalculated based on the sums of remaining transcripts.
>
> You can compare the results from using total number of reads and number of mapped reads. In my experience, I have detected little to no difference in the identified DE genes (not for the subset of genes I was investigating). Do keep in mind that a different library size will lead to different levels of normalization.

2. Create design matrix

A design matrix is a way of explicitly coding your experimental/sampling design into a form that can be used by a statistical model in R. You can choose to designate a grouping variable to each sample during the creation of the DGEList by adding the filling the `group` argument. However, that is only useful in simple multi-group comparisons (analogous to one-way ANOVA). A design matrix allows us to flexibly partition data in many different ways to test different hypotheses akin to specifying predictors in generalized linear models. 

In my experience, it is better to explicitly specify the design matrix (regardless of design complexity) so we can track what is being tested, especially when there is more than 2 groups or a mix of predictor data types (categorical, quantitative, ordinal, nominal). Below are just some examples using categorical data:

```R
# One-way comparisons
design <- with(dge, model.matrix(~ 0 + groups, samples))

# Paired-sample design
design <- with(dge, model.matrix(~ group1 + group2, samples))

# Factorial design
design <- with(dge, model.matrix(~ group1 + group2 + group1:group2, samples))

# Nested interactions
design <- with(dge, model.matrix(~ group1 + group1:group2, samples))
```

3. Filter count data

Count data can be sparse. This is not useful for testing DE genes. `edgeR` can filter them out accordingly while respecting the design matrix.

```R
keep <- filterByExpr(dge, design = design)
summary(keep) # Check how many genes remain for downstream analysis
dge <- dge[keep, , keep.lib.sizes = T]
```

4. Calculate TMM-normalization factors
   
```R
dge <- calcNormFactors(dge)
```

5. Estimate negative binomial dispersions

```R
# Estimate dispersions
dge <- estimateDisp(dge, design = design, robust = T)
# Visualize dispersions
plotBCV(dge) 
```

6. Fit model to data

```R
fit <- glmQLFit(dge, design = design, robust = T)
``` 

We should also inspect how biological variation and dispersion estimates change across the data.

```R
par(mfrow = c(1, 2))
plotBCV(dge)
plotQLDisp(fit)
```

7. Test genes against hypotheses and extract DE genes

Depending on your design matrix and study questions, this step can be quite interactive and iterative. Because of the flexibility built into `edgeR`, we can test for any number of contrasts and perform overall tests (think ANOVA F-test). 

For simplicity, let's assume in the first example that the experiment is a simple comparison between 2 groups with a model formulation: `~ group`

```R
# Test for DE genes
qlf <- glmQLFTest(fit)
# Check how many genes were significantly down- or up-regulated
summary(decideTests(qlf, p.value = 0.05))
# Obtain a table of significantly DE genes and associated annotations/features
ttg <- topTags(qlf, n = Inf)
```

In a slightly more complicated example, we will assume the experiment requires pairwise comparison between 3 groups (A, B, and C) with a model formulation: `~ 0 + group`. We begin by creating a contrast matrix for ease of accessing comparisons. Then we test for overall effects, then extract contrasts of interest.

```R
my_contrasts <- makeContrasts(
  A.vs.B = A - B,
  A.vs.C = A - C,
  B.vs.C = B - C,
  levels = design
)
# The above creates a matrix with contrasts in each column and groups in each row 

# Overall test for DE genes
qlf_overall <- glmQLFTest(fit, contrast = my_contrasts)
summary(decideTests(qlf_overall, p.value = 0.05))
ttg_overall <- topTags(qlf_overall, n = Inf)

# DE genes between groups A and C
qlf_AC <- glmQLFTest(fit, contrast = my_contrasts[, "A.vs.C"])
summary(decideTest(qlf_AC, p.value = 0.05))
ttg_AC <- topTags(qlf_AC, n = Inf)
## Change the contrast argument for different contrasts of interest

```

8. Plot group contrasts of interest

Following from the comparison between groups A and C, we can visualize the results by plotting a mean-difference plot or a volcano plot.

```R
# Mean-Difference plot
plotMD(qlf_AC)

# Volcano plot
as.data.frame(ttg_AC) %>%
  mutate(
    col = case_when(
      FDR < 0.05 & sign(logFC) > 0 ~ "Up",
      FDR < 0.05 & sign(logFC) < 0 ~ "Down",
      T ~ "Not Sig"
    ) %>%
      factor(., levels = c("Up", "Not Sig", "Down"))
  ) %>%
  ggplot(aes(x = logFC, y = -log10(PValue), colour = col)) +
    geom_point() +
    scale_colour_manual(name = NULL, values = c("red", "grey80", "blue")) +
    theme_light() +
    theme(panel.grid = element_blank())
```

**Obtaining normalized count data**

You can extract RPKM values from the DGEList object created using edgeR. You can also opt to use normalized library sizes (TMM by default) to calculate RPKM. This can then be used to calculate TPM values for inspecting trends on plots. Make sure you have gene length information during the creating of the DGEList object before proceeding.

```R
RPKM <- rpkm(dge, normalized.lib.sizes = T, log = F)
TPM <- t(t(RPKM)/colSums(RPKM)) * 1e6
```

### DESeq2


### limma-voom
