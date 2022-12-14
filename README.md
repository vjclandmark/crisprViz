crisprViz: visualization of CRISPR guide RNAs (gRNAs)
================

-   [Introduction](#introduction)
-   [Installation and getting
    started](#installation-and-getting-started)
    -   [Software requirements](#software-requirements)
        -   [OS Requirements](#os-requirements)
    -   [Installation](#installation)
-   [Use cases](#use-cases)
    -   [Visualizing the best gRNAs for a given
        gene](#visualizing-the-best-grnas-for-a-given-gene)
    -   [Plotting for precision
        targeting](#plotting-for-precision-targeting)
    -   [CRISPRa and adding genomic
        annotations](#crispra-and-adding-genomic-annotations)
    -   [Comparing multiple GuideSets targeting the same
        region](#comparing-multiple-guidesets-targeting-the-same-region)
-   [Setting plot size](#setting-plot-size)
-   [Session Info](#session-info)

Author: Luke Hoberecht

Date: July 24, 2022

# Introduction

The `crisprViz` package enables the graphical interpretation of
`GuideSet` objects from the `crisprDesign` package by plotting guide RNA
(gRNA) cutting locations against their target gene or other genomic
region. These genomic plots are constructed using the `Gviz` package
from Bioconductor.

This vignette walks through several use cases that demonstrate the range
of and how to use plotting functions in the `crisprViz` package. This
vignette also makes heavy use of the `crisprDesign` package to
manipulate `GuideSet` objects in conjunction with plotting in the
process of gRNA design. For more information about the `crisprDesign`
package see \[vignettes\].

# Installation and getting started

## Software requirements

### OS Requirements

This package is supported for macOS, Linux and Windows machines.
Packages were developed and tested on R version 4.2.

## Installation

`crisprViz` and its dependencies can be installed by typing the
following commands inside of an R session:

``` r
install.packages("devtools")
devtools::install_github("Jfortin1/crisprBase")
devtools::install_github("Jfortin1/crisprDesign")
devtools::install_github("Jfortin1/crisprViz")
```

# Use cases

All examples in this vignette will use human genome assembly `hg38` from
the `BSgenome.Hsapiens.UCSC.hg38` package and gene model coordinates
from Ensembl release 104. We begin by loading the necessary packages.

``` r
library(BSgenome.Hsapiens.UCSC.hg38)
library(crisprDesign)
library(crisprViz)
```

## Visualizing the best gRNAs for a given gene

Suppose we want to design the four best gRNAs using the SpCas9 CRISPR
nuclease to knockout the human KRAS gene. To have the greatest impact on
gene function we want to prioritize gRNAs that have greater isoform
coverage, and target closer to the 5??? end of the CDS.

Let???s load a precomputed `GuideSet` object containing all possible gRNAs
targeting the the CDS of KRAS, and a `GRangesList` object describing the
gene model for KRAS.

``` r
data("krasGuideSet", package="crisprViz")
data("krasGeneModel", package="crisprViz")
length(krasGuideSet) # number of candidate gRNAs
```

    ## [1] 52

For how to design such gRNAs, see the `crisprDesign` package. Before we
plot all of our candidate gRNAs, let???s first generate a simple plot with
a few gRNAs to familiarize ourselves with some plot components and
options.

``` r
plotGuideSet(krasGuideSet[1:4],
             geneModel=krasGeneModel,
             targetGene="KRAS")
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

There are a few things to note here.

-   The ideogram track and genome axis track are at the top of our plot
    and give us coordinate information.
-   Our `targetGene` KRAS is plotted next, using coordinates from the
    provided gene model `krasGeneModel`, followed by our spacer subset.
    The name of each track is given on the left.
-   The strand information for each track is included in the label: `<-`
    for reverse strand and `->` for forward strand.
-   While we can identify which exon each spacer targets (which may be
    sufficient), the plot window is too large to provide further
    information.
-   The plot only shows the 3??? end of KRAS, rather than the entire gene.

This last point is important: the default plot window is set by the
spacers??? ranges in the input `GuideSet` object. We can manually adjust
this window by using the `from`, `to`, `extend.left`, and `extend.right`
arguments. Here is the same plot adjusted to show the whole KRAS gene,
which also reveals an additional isoform that is not targeted by any
spacer in this example subset.

``` r
from <- min(start(krasGeneModel$transcripts))
to <- max(end(krasGeneModel$transcripts))
plotGuideSet(krasGuideSet[1:4],
             geneModel=krasGeneModel,
             targetGene="KRAS",
             from=from,
             to=to,
             extend.left=1000,
             extend.right=1000)
```

![](README_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

As calculated above, there are a total of 52 candidate gRNAs targeting
the CDS of KRAS. Including all of them could crowd the plot space,
making it difficult to interpret. To alleviate this we can hide the gRNA
labels by setting the `showGuideLabels` argument to `FALSE`.

``` r
plotGuideSet(krasGuideSet,
             geneModel=krasGeneModel,
             targetGene="KRAS",
             showGuideLabels=FALSE,
             from=from,
             to=to,
             extend.left=1000,
             extend.right=1000)
```

![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

At the gene level, the plot window is too large to discern details for
each spacer target. However, we can see five distinct clusters of spacer
target locations that cover the CDS of KRAS. The spacers in the 5???-most
cluster (on the reverse strand) target the only coding region of the
gene that is expressed by all isoforms, making it an ideal target for
our scenario.

We can see which gRNAs target this region by returning `showGuideLabels`
to its default value of `TRUE`, and by adjusting the plot window to
focus on our exon of interest.

``` r
# new window range around target exon
targetExon <- queryTxObject(krasGeneModel,
                            featureType="cds",
                            queryColumn="exon_id",
                            queryValue="ENSE00000936617")
targetExon <- unique(targetExon)
from <- start(targetExon)
to <- end(targetExon)
plotGuideSet(krasGuideSet,
             geneModel=krasGeneModel,
             targetGene="KRAS",
             from=from,
             to=to,
             extend.left=20,
             extend.right=20)
```

![](README_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

At this resolution we can get a much better idea of spacer location and
orientation. In particular, the PAM sequence is visible as a narrow box
on the 3??? side of our protospacer sequences. We can also distinctly see
which spacer targets overlap each other???it may be best to avoid pairing
such spacers in some applications lest they sterically interfere with
each other.

If we have many gRNA targets in a smaller window and are not concerned
with overlaps, we can configure the plot to only show the `pam_site`,
rather than the entire protospacer and PAM sequence, by setting
`pamSiteOnly` to `TRUE`.

``` r
plotGuideSet(krasGuideSet,
             geneModel=krasGeneModel,
             targetGene="KRAS",
             from=from,
             to=to,
             extend.left=20,
             extend.right=20,
             pamSiteOnly=TRUE)
```

![](README_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

Let???s filter our `GuideSet` by the spacer names in the plot then pass an
on-target score column in our `GuideSet` to `onTargetScores` to color
the spacers according to that score, with darker blue colors indicating
higher scores. Note that for this plot we need not provide values for
`from` and `to`, as the plot window adjusts to our filtered `GuideSet`.

``` r
selectedGuides <- c("spacer_80", "spacer_84", "spacer_88", "spacer_92",
                    "spacer_96", "spacer_100", "spacer_104", "spacer_108",
                    "spacer_112")
candidateGuides <- krasGuideSet[selectedGuides]
plotGuideSet(candidateGuides,
             geneModel=krasGeneModel,
             targetGene="KRAS",
             onTargetScore="score_deephf")
```

![](README_files/figure-gfm/unnamed-chunk-9-1.png)<!-- -->

<!-- ![legend caption](file path) -->

## Plotting for precision targeting

For a given CRISPR application, the target region may consist of only
several base pairs rather than an exon or entire gene CDS. In these
instances it may be important to know exactly where the gRNAs target,
and plots of gRNAs must be at a resolution capable of distinguishing
individual bases. This is often the case for CRISPR base editor
(CRISPRbe) applications, as the editing window for each gRNA is narrow
and the results are specific to each target sequence.

In this example, we will zoom in on a few gRNAs targeting the 5??? end of
the human GPR21 gene. We want our plot to include genomic sequence
information so we will set the `bsgenome` argument to the same
`BSgenome` object we used to create our `GuideSet`.

First, we load the precomputed `GuideSet` and gene model objects for
GPR21,

``` r
data("gpr21GuideSet", package="crisprViz")
data("gpr21GeneModel", package="crisprViz")
```

and then plot the gRNAs.

``` r
plotGuideSet(gpr21GuideSet,
             geneModel=gpr21GeneModel,
             targetGene="GPR21",
             bsgenome=BSgenome.Hsapiens.UCSC.hg38,
             margin=0.3)
```

![](README_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

The genomic sequence is given at the bottom of the plot as color-coded
boxes. The color scheme for describing the nucleotides is given in the
[**biovizBase**
package](https://bioconductor.org/packages/3.16/bioc/html/biovizBase.html).
If the plot has sufficient space, it will display nucleotide symbols
rather than boxes. We can accomplish this by plotting a narrower range
or by increasing the width of our plot space (see ???Setting plot size???
section).

The plot above was generated with a plot space width of 6 inches; here???s
the same plot after we increase the width to 10 inches:

``` r
# increase plot width from 6" to 10"
plotGuideSet(gpr21GuideSet,
             geneModel=gpr21GeneModel,
             targetGene="GPR21",
             bsgenome=BSgenome.Hsapiens.UCSC.hg38,
             margin=0.3)
```

![](README_files/figure-gfm/unnamed-chunk-12-1.png)<!-- -->

## CRISPRa and adding genomic annotations

In this scenario we want to increase expression of the human MMP7 gene
via CRISPR activation (CRISPRa). We will use the SpCas9 CRISPR nuclease.

``` r
data("mmp7GuideSet", package="crisprViz")
data("mmp7GeneModel", package="crisprViz")
```

The `GuideSet` contains candidate gRNAs in the 2kb window immediately
upstream of the TSS of MMP7. We will also use a `GRanges` object
containing repeat elements in this region:

``` r
data("repeats", package="crisprViz")
```

Let???s begin by plotting our `GuideSet`, and adding a track of repeat
elements using the `annotations` argument. Our `guideSet` also contains
SNP annotation, which we would also prefer our gRNAs to not overlap. To
include a SNP annotation track, we will set `includeSNPTrack=TRUE`
(default).

``` r
from <- min(start(mmp7GuideSet))
to <- max(end(mmp7GuideSet))
plotGuideSet(mmp7GuideSet,
             geneModel=mmp7GeneModel,
             targetGene="MMP7",
             guideStacking="dense",
             annotations=list(Repeats=repeats),
             pamSiteOnly=TRUE,
             from=from,
             to=to,
             extend.left=600,
             extend.right=100,
             includeSNPTrack=TRUE)
```

![](README_files/figure-gfm/unnamed-chunk-15-1.png)<!-- -->

Some of our candidate gRNAs target repeat elements and likely target a
large number of loci in the genome, potentially causing unintended
effects, or overlap with SNPs, which can reduce its activity. Let???s
remove these gRNAs and regenerate the plot.

``` r
filteredGuideSet <- crisprDesign::removeRepeats(mmp7GuideSet,
                                                gr.repeats=repeats)
filteredGuideSet <- crisprDesign::filterSpacers(filteredGuideSet,
                                                criteria=list(hasSNP=FALSE))
plotGuideSet(filteredGuideSet,
             geneModel=mmp7GeneModel,
             targetGene="MMP7",
             guideStacking="dense",
             annotations=list(Repeats=repeats),
             pamSiteOnly=TRUE,
             from=from,
             to=to,
             extend.left=600,
             extend.right=100,
             includeSNPTrack=TRUE)
```

![](README_files/figure-gfm/unnamed-chunk-16-1.png)<!-- -->

Note how removing gRNAs that overlap SNPs from our `GuideSet` also
removed the SNP track. To prevent plotting an empty track,
`plotGuideSet` will only include a SNPs track if at least one gRNA
includes SNP annotation (i.e.??overlaps a SNP).

Conversely, there are specific genomic regions that would be beneficial
to target, such as CAGE peaks and DNase I Hypersensitivity tracks. We
show in the `inst\scripts` folder how to obtain such data from the
Bioconductor package `AnnotationHub`, but for the sake of time, we have
precomputed those objects and they can be loaded from the `crisprViz`
package directly:

``` r
data("cage", package="crisprViz")
data("dnase", package="crisprViz")
```

We now plot gRNAs alongside with those two tracks:

``` r
plotGuideSet(filteredGuideSet,
             geneModel=mmp7GeneModel,
             targetGene="MMP7",
             guideStacking="dense",
             annotations=list(CAGE=cage, DNase=dnase),
             pamSiteOnly=TRUE,
             from=from,
             to=to,
             extend.left=600,
             extend.right=100)
```

![](README_files/figure-gfm/unnamed-chunk-18-1.png)<!-- -->

Let???s filter our `GuideSet` for guides overlapping the plotted DNase
site then regenerate the plot.

``` r
# filter GuideSet for gRNAs overlapping DNase track
overlaps <- findOverlaps(filteredGuideSet, dnase, ignore.strand=TRUE)
finalGuideSet <- filteredGuideSet[queryHits(overlaps)]
plotGuideSet(finalGuideSet,
             geneModel=mmp7GeneModel,
             targetGene="MMP7",
             guideStacking="dense",
             annotations=list(CAGE=cage, DNase=dnase),
             pamSiteOnly=TRUE,
             margin=0.4)
```

![](README_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

## Comparing multiple GuideSets targeting the same region

The choice of the CRISPR nuclease can be influenced by the abundance of
PAM sequences recognized by a given nuclease in the target region. For
example, we would expect AT-rich regions to have fewer possible targets
for the SpCas9 nuclease, whose PAM is NGG. In these regions, the CRISPR
nuclease AsCas12a, whose PAM is TTTV, may prove more appropriate. Given
multiple `GuideSet`s targeting the same region, we can compare the gRNAs
of each in the same plot using `plotMultipleGuideSets`.

Here, we pass our `GuideSet`s targeting an exon in the human gene LTN1
in a named list. Note that there are no available options for displaying
guide labels or guide stacking, and only the PAM sites are plotted. We
will also add a track to monitor the percent GC content (using a window
roughly the length of our protospacers). Not surprisingly, this AT-rich
region has fewer targets for SpCas9 compared to AsCas12a. (Note: when
plotting several GuideSets you may need to increase the height of the
plot space in order for the track names to appear on the left side; see
???Setting plot size??? below.)

We first load the precomputed `GuideSet` objects:

``` r
data("cas9GuideSet", package="crisprViz")
data("cas12aGuideSet", package="crisprViz")
data("ltn1GeneModel", package="crisprViz")
```

``` r
plotMultipleGuideSets(list(SpCas9=cas9GuideSet, AsCas12a=cas12aGuideSet),
                      geneModel=ltn1GeneModel,
                      targetGene="LTN1",
                      bsgenome=BSgenome.Hsapiens.UCSC.hg38,
                      margin=0.2,
                      gcWindow=10)
```

![](README_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

# Setting plot size

Plots with many gene isoforms and/or gRNAs may require more space to
render than is allotted by your graphical device???s default settings,
resulting in an error. One solution, depending on your graphical device,
is offered by the [**grDevices**
package](https://www.rdocumentation.org/packages/grDevices/versions/3.6.2/topics/Devices).

Here is an example using macOS Quartz device:

``` r
grDevices::quartz("Example plot", width=6, height=7)
# plot function
```

# Session Info

``` r
sessionInfo()
```

    ## R Under development (unstable) (2022-03-21 r81954)
    ## Platform: x86_64-apple-darwin17.0 (64-bit)
    ## Running under: macOS Catalina 10.15.7
    ## 
    ## Matrix products: default
    ## BLAS:   /Library/Frameworks/R.framework/Versions/4.2/Resources/lib/libRblas.0.dylib
    ## LAPACK: /Library/Frameworks/R.framework/Versions/4.2/Resources/lib/libRlapack.dylib
    ## 
    ## locale:
    ## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
    ## 
    ## attached base packages:
    ## [1] stats4    stats     graphics  grDevices utils     datasets  methods  
    ## [8] base     
    ## 
    ## other attached packages:
    ##  [1] crisprViz_0.99.14                 crisprDesign_0.99.109            
    ##  [3] crisprBase_1.1.2                  BSgenome.Hsapiens.UCSC.hg38_1.4.4
    ##  [5] BSgenome_1.63.5                   rtracklayer_1.55.4               
    ##  [7] Biostrings_2.63.2                 XVector_0.35.0                   
    ##  [9] GenomicRanges_1.47.6              GenomeInfoDb_1.31.6              
    ## [11] IRanges_2.29.1                    S4Vectors_0.33.11                
    ## [13] BiocGenerics_0.41.2              
    ## 
    ## loaded via a namespace (and not attached):
    ##   [1] backports_1.4.1               Hmisc_4.7-0                  
    ##   [3] AnnotationHub_3.3.9           BiocFileCache_2.3.4          
    ##   [5] lazyeval_0.2.2                splines_4.2.0                
    ##   [7] BiocParallel_1.29.18          ggplot2_3.3.5                
    ##   [9] digest_0.6.29                 ensembldb_2.19.10            
    ##  [11] htmltools_0.5.2               fansi_1.0.2                  
    ##  [13] checkmate_2.0.0               magrittr_2.0.2               
    ##  [15] memoise_2.0.1                 cluster_2.1.2                
    ##  [17] tzdb_0.2.0                    readr_2.1.2                  
    ##  [19] matrixStats_0.61.0            prettyunits_1.1.1            
    ##  [21] jpeg_0.1-9                    colorspace_2.0-3             
    ##  [23] blob_1.2.2                    rappdirs_0.3.3               
    ##  [25] crisprScoreData_1.1.3         xfun_0.30                    
    ##  [27] dplyr_1.0.8                   crayon_1.5.0                 
    ##  [29] RCurl_1.98-1.6                jsonlite_1.8.0               
    ##  [31] survival_3.3-1                VariantAnnotation_1.41.3     
    ##  [33] glue_1.6.2                    gtable_0.3.0                 
    ##  [35] zlibbioc_1.41.0               DelayedArray_0.21.2          
    ##  [37] scales_1.1.1                  DBI_1.1.2                    
    ##  [39] Rcpp_1.0.8.3                  xtable_1.8-4                 
    ##  [41] progress_1.2.2                htmlTable_2.4.0              
    ##  [43] reticulate_1.25               foreign_0.8-82               
    ##  [45] bit_4.0.4                     Formula_1.2-4                
    ##  [47] htmlwidgets_1.5.4             httr_1.4.2                   
    ##  [49] dir.expiry_1.3.0              RColorBrewer_1.1-2           
    ##  [51] ellipsis_0.3.2                pkgconfig_2.0.3              
    ##  [53] XML_3.99-0.9                  Gviz_1.39.5                  
    ##  [55] nnet_7.3-17                   dbplyr_2.1.1                 
    ##  [57] utf8_1.2.2                    tidyselect_1.1.2             
    ##  [59] rlang_1.0.2                   later_1.3.0                  
    ##  [61] AnnotationDbi_1.57.1          munsell_0.5.0                
    ##  [63] BiocVersion_3.15.0            tools_4.2.0                  
    ##  [65] cachem_1.0.6                  cli_3.3.0                    
    ##  [67] generics_0.1.2                RSQLite_2.2.12               
    ##  [69] ExperimentHub_2.3.5           evaluate_0.15                
    ##  [71] stringr_1.4.0                 fastmap_1.1.0                
    ##  [73] yaml_2.3.5                    knitr_1.37                   
    ##  [75] bit64_4.0.5                   purrr_0.3.4                  
    ##  [77] randomForest_4.7-1            AnnotationFilter_1.19.0      
    ##  [79] KEGGREST_1.35.0               Rbowtie_1.35.0               
    ##  [81] mime_0.12                     xml2_1.3.3                   
    ##  [83] biomaRt_2.51.3                compiler_4.2.0               
    ##  [85] rstudioapi_0.13               filelock_1.0.2               
    ##  [87] curl_4.3.2                    png_0.1-7                    
    ##  [89] interactiveDisplayBase_1.33.0 tibble_3.1.6                 
    ##  [91] stringi_1.7.6                 crisprScore_1.1.13           
    ##  [93] highr_0.9                     basilisk.utils_1.9.1         
    ##  [95] GenomicFeatures_1.47.13       lattice_0.20-45              
    ##  [97] ProtGenerics_1.27.2           Matrix_1.4-0                 
    ##  [99] vctrs_0.3.8                   pillar_1.7.0                 
    ## [101] lifecycle_1.0.1               BiocManager_1.30.16          
    ## [103] data.table_1.14.2             bitops_1.0-7                 
    ## [105] httpuv_1.6.5                  R6_2.5.1                     
    ## [107] BiocIO_1.5.0                  latticeExtra_0.6-29          
    ## [109] promises_1.2.0.1              gridExtra_2.3                
    ## [111] dichromat_2.0-0               crisprBowtie_1.1.1           
    ## [113] assertthat_0.2.1              SummarizedExperiment_1.25.3  
    ## [115] rjson_0.2.21                  GenomicAlignments_1.31.2     
    ## [117] Rsamtools_2.11.0              GenomeInfoDbData_1.2.7       
    ## [119] parallel_4.2.0                hms_1.1.1                    
    ## [121] grid_4.2.0                    rpart_4.1.16                 
    ## [123] basilisk_1.9.2                rmarkdown_2.13               
    ## [125] MatrixGenerics_1.7.0          biovizBase_1.43.1            
    ## [127] Biobase_2.55.0                shiny_1.7.1                  
    ## [129] base64enc_0.1-3               restfulr_0.0.13
