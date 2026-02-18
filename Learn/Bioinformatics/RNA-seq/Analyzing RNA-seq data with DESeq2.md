### Based on
[Analyzing RNA-seq data with DESeq2
](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html)

***

## Input Data
DESeq2 model internally corrects for library size, so transformed or normalized values such as counts scaled by library size should not be used as input.

$~$

#### The DESeqDataSet `dds`
    In code,`dds` store the read counts and the intermediate estimated quantities.

    DESeqDataSet class extends the RangedSummarizedExperiment class of the SummarizedExperiment package.

A DESeqDataSet object must have an associated design formula to expresses the variables which will be used in modeling. 

The design formula is involved to estimate the dispersions and to estimate the log2 fold changes of the model.

$~$

##### DESeqDataSet Construct - Transcript abundance files and tximport/tximeta

Recommended pipeline:
1. Fast transcript abundance quantifiers
2. DESeq2 to create gene-level count matrices (Use tximport for data import)

Transcript abundance estimation under this method will:
1. Corrects for potential changes in gene length (isoform) across samples
2. Faster and require less memory and disk usage compared to alignment-based methods
3. Avoid discarding those fragments that can align to multiple genes with homologous sequence
4. To obtain transcript abundance estimation:

    [Salmon](https://combine-lab.github.io/salmon/getting_started/)

$~$

##### Transcript abundance import and gene-level DESeqDataSet object construct from Salmon `quant.sf`
```[R]
library("tximport")
library("readr")
library("tximportData")
dir <- system.file("extdata", package="tximportData")
samples <- read.table(file.path(dir,"samples.txt"), header=TRUE)
samples$condition <- factor(rep(c("A","B"),each=3))
rownames(samples) <- samples$run
samples[,c("pop","center","run","condition")]
```
```
##           pop center       run condition
## ERR188297 TSI  UNIGE ERR188297         A
## ERR188088 TSI  UNIGE ERR188088         A
## ERR188329 TSI  UNIGE ERR188329         A
## ERR188288 TSI  UNIGE ERR188288         B
## ERR188021 TSI  UNIGE ERR188021         B
## ERR188356 TSI  UNIGE ERR188356         B
```
```
## specify the path to the file
files <- file.path(dir,"salmon", samples$run, "quant.sf.gz")
names(files) <- samples$run
tx2gene <- read_csv(file.path(dir, "tx2gene.gencode.v27.csv"))
```
If dir is `/path/to/extdata` and `samples$run` contains `ERR188297`, one of the file paths in files will be: `/path/to/extdata/salmon/ERR188297/quant.sf.gz`

```
txi <- tximport(files, type="salmon", tx2gene=tx2gene)

## Construct DESeqDataSet
## objects from txi
## sample information from samples

library("DESeq2")
ddsTxi <- DESeqDataSetFromTximport(txi,
                                   colData = samples,
                                   design = ~ condition)
```
$~$

#####  Transcript abundance import from Tximeta with automatic metadata
Tximeta (Love et al. 2020) extends tximport, offering the same functionality. Additional benefit is automatic addition of annotation metadata for commonly used transcriptomes.

Tximeta produces a S`ummarizedExperiment` that can be loaded into DESeq2.
```
coldata <- samples
coldata$files <- files
coldata$names <- coldata$run

## Loads the tximeta package, which automates transcript-to-gene mapping and adds annotation metadata
library("tximeta") 

## Reads the transcript abundance data from the Salmon files
## Maps transcript-level counts to genes
## Returns a SummarizedExperiment object containing gene-level counts and automatic metadata
se <- tximeta(coldata)

##Convert SummarizedExperiment into DESeqDataSet with a design formula based on the condition variable
ddsTxi <- DESeqDataSet(se, design = ~ condition)
```

$~$
#### Count matrix input