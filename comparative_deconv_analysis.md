``` r
library(nanostringr)
library(survival)
library(survminer)
```

    ## Loading required package: ggplot2

    ## Loading required package: ggpubr

    ## 
    ## Attaching package: 'survminer'

    ## The following object is masked from 'package:survival':
    ## 
    ##     myeloma

``` r
library(ggplot2)
library(ggpubr)
library(ComplexHeatmap)
```

    ## Loading required package: grid

    ## ========================================
    ## ComplexHeatmap version 2.14.0
    ## Bioconductor page: http://bioconductor.org/packages/ComplexHeatmap/
    ## Github page: https://github.com/jokergoo/ComplexHeatmap
    ## Documentation: http://jokergoo.github.io/ComplexHeatmap-reference
    ## 
    ## If you use it in published research, please cite either one:
    ## - Gu, Z. Complex Heatmap Visualization. iMeta 2022.
    ## - Gu, Z. Complex heatmaps reveal patterns and correlations in multidimensional 
    ##     genomic data. Bioinformatics 2016.
    ## 
    ## 
    ## The new InteractiveComplexHeatmap package can directly export static 
    ## complex heatmaps into an interactive Shiny app with zero effort. Have a try!
    ## 
    ## This message can be suppressed by:
    ##   suppressPackageStartupMessages(library(ComplexHeatmap))
    ## ========================================

``` r
library(cluster)
library(readxl)
library(MCPcounter)
```

    ## Loading required package: curl

    ## Using libcurl 8.7.1 with LibreSSL/3.3.6

# Read in COTC021/22 bulk nanostring data

``` r
rcc_dir <- list.dirs("~/Dog/Nanostring/", recursive = FALSE)
raw_expr <- lapply(rcc_dir, function(x) read_rcc(path = x)$raw)
sfs <- list()
sfs2 <- list()
i = 1
norm_expr <- lapply(raw_expr, function(x) {
  
  #positive control normalization
  gm <- exp(colMeans(log(x[x$Code.Class == "Positive",4:15]), na.rm = T))
  am <- mean(gm, na.rm = T)
  sf <- sapply(gm, function(y) am/y)
  sfs[[i]] <<- sf
  x1 <- as.matrix(x[,4:15])
  x1 <- sapply(1:ncol(x1), function(i) sf[i] * as.numeric(x1[,i]))
  
  #House keeping gene normalization
  gm <- exp(colMeans(log(x[x$Code.Class == "Housekeeping", 4:15]), na.rm = T))
  am <- mean(gm, na.rm = T)
  sf <- sapply(gm, function(y) am/y)
  sfs2[[i]] <<- sf
  x2 <- sapply(1:ncol(x1),function(i) sf[i]*as.numeric(x1[,i]))
  rownames(x2) <- x$Name
  colnames(x2) <- colnames(x)[4:ncol(x)]
  x2 <- x2[x$Code.Class == "Endogenous",]
  x2 <- x2[,!is.na(sf) & !is.infinite(sf)]
  i <<- i+1
  return(x2)
})

combined_expr <- Reduce(cbind, norm_expr)
rownames(combined_expr) <- rownames(norm_expr[[1]])
samp_names <- sapply(colnames(combined_expr), function(x) {
splitstr = strsplit(x,split = "_")[[1]]
return(splitstr[3])
})
samp_names <- sapply(samp_names, trimws)
colnames(combined_expr) <- samp_names
combined_expr <- combined_expr[,!grepl("0814 RNAlater tumor", samp_names)]
nanostring_annotations <- read_excel("~/Dog/Nanostring/CS032282 and CS9318963 Nanostring 07-29-2022.xlsx")
nanostring_annotations <- as.data.frame(nanostring_annotations)
rownames(nanostring_annotations) <- nanostring_annotations$`Nanostring Sample Name`
common <- intersect(rownames(nanostring_annotations), colnames(combined_expr))
CE <- combined_expr[,common]
NaAn <- nanostring_annotations[common,]
```

# Read COTC021/22 bulk RNA Seq data

``` r
library(readr)
```

    ## 
    ## Attaching package: 'readr'

    ## The following object is masked from 'package:curl':
    ## 
    ##     parse_date

``` r
Canine_RNA_SEQ_TPM_All_data <- read_csv("~/Downloads/Canine_RNA_SEQ_TPM_All_data.csv")
```

    ## Rows: 37952 Columns: 199

    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr   (1): gene_id
    ## dbl (198): 100_0514_tumor_rsem.genes.results, 101_0518_tumor_rsem.genes.resu...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
Canine_RNA_SEQ_TPM_All_data <- as.data.frame(Canine_RNA_SEQ_TPM_All_data)
rownames(Canine_RNA_SEQ_TPM_All_data) <- Canine_RNA_SEQ_TPM_All_data$gene_id
Canine_RNA_SEQ_TPM_All_data <- Canine_RNA_SEQ_TPM_All_data[,-1]
sample_type <- c("Met","Primary")[as.numeric(grepl("(Tumor)|(tumor)", colnames(Canine_RNA_SEQ_TPM_All_data)))+1]
colnames(Canine_RNA_SEQ_TPM_All_data) <- sapply(colnames(Canine_RNA_SEQ_TPM_All_data), function(x) as.character(as.numeric(strsplit(x,split = '_')[[1]][2])))
rownames(Canine_RNA_SEQ_TPM_All_data) <- sapply(rownames(Canine_RNA_SEQ_TPM_All_data), function(x) gsub("_3","",x))

DOG2_RNAseq_Sample_list <- read_excel("~/Downloads/DOG2 RNAseq Sample list.xlsx")
DOG2_RNAseq_Sample_list$`Sample ID`[7] <- "18_0620_Met_Skin"
DOG2_RNAseq_Sample_list$`Sample ID`[8] <- "20_0621_Met_Unspecified"
sites_RNASeq <- rep("Primary", ncol(Canine_RNA_SEQ_TPM_All_data))
sites_RNASeq[sample_type == "Met"] <- sapply(DOG2_RNAseq_Sample_list$`Sample ID`[1:8], function(x) toupper(strsplit(x,split = "_")[[1]][4]))
```

# helper functions to measure expression of distinct pathway signatures: (eg chromosomal instability, cGAS-STING, type1 interferon, etc)

``` r
library(msigdbr)
msigdbr_df <- msigdbr(species = "Homo sapiens", category = "H")
msigdbr_list = split(x = msigdbr_df$gene_symbol, f = msigdbr_df$gs_name)
CIN_signature <- function(expr){
    genes <- c("CCNB2","PRC1","CDC45L","KIF20A","UBE2C","CDC2","MAD2L1","H2AFZ","TRIP13","ESPL1","TTK","FEN1","MCM7","CNAP1","MELK","TPX2","FOXM1","TOP2A","MCM2","RAD51AP1","RFC4","PCNA","RNASEH2A","TGIF2","CCT5")
    mat <- as.data.frame(expr)
    mat <- mat[genes,]
    mat <- apply(as.matrix(mat),1,scale)
    scores <- rowMeans(mat, na.rm = T)
    return(scores)
}

CYT_signature <- function(expr){
    genes <- c("GZMA","PRF1")
    mat <- as.data.frame(expr)
    mat <- log2(mat[genes,] +1)
    scores <- colMeans(mat, na.rm = T)
    return(scores)
}

type1_interferon_signature <- function(expr) {
  genes <- msigdbr_list$HALLMARK_INTERFERON_ALPHA_RESPONSE
    mat <- as.data.frame(expr)
    mat <- mat[genes,]
    mat <- apply(as.matrix(mat),1,scale)
    scores <- rowMeans(mat, na.rm = T)
    return(scores)
}

cGAS_STING_signature <- function(expr){
  genes = c("CGAS","STING1","TBK1","IKBKE","IRF3")
  mat <- as.data.frame(expr)
  mat <- mat[genes,]
  mat <- apply(as.matrix(mat),1,scale)
  scores <- rowMeans(mat, na.rm = T)
  return(scores)
}
```

# run MCP counter + clustering analysis on bulk rnaseq data

``` r
library(RColorBrewer)
set.seed(2425)
canin_exp <- Canine_RNA_SEQ_TPM_All_data



cellprop_rnaseq<- t(MCPcounter.estimate(expression = log2(canin_exp+1), featuresType = "HUGO_symbols", probesets = read.table("~/Dog/MCP_counter_markers.csv")))
cellprop_rnaseq <- apply(cellprop_rnaseq, 2, scale)
rownames(cellprop_rnaseq) <- colnames(canin_exp)

#k means clustering
km.res <- kmeans(cellprop_rnaseq, centers = 3, nstart = 500, iter.max = 500)
cls <- km.res$cluster


imm_clus <- rep("ID", nrow(cellprop_rnaseq))
imm_clus[cls == 1] <- "IE-ECM"
imm_clus[cls == 2] <- "IE"
ii1 <- imm_clus
imm_clus <- factor(imm_clus, levels = c("IE","IE-ECM","ID"))


#Plot results
ha = rowAnnotation(
    sample = sample_type,
    preservation = rep("Frozen", length(sample_type)),
    col = list("sample" = c("Primary" = "yellow", "Met" =  "darkgreen"),
               "preservation" = c("Frozen" = "#db6d00", "FFPE" = "#924900")
    )
)

n <- 45
qual_col_pals = brewer.pal.info[brewer.pal.info$category == 'qual',]
col_vector = unlist(mapply(brewer.pal, qual_col_pals$maxcolors, rownames(qual_col_pals)))
cols <- sample(col_vector, n, replace = FALSE)
cols <- sample(cols, length(unique(sites_RNASeq)), replace = F)
names(cols) <- unique(sites_RNASeq)
cols["Primary"] <- "white"
ha2 = rowAnnotation(
    site = sites_RNASeq,
    col = list("site"=cols)
)
Heatmap(cellprop_rnaseq, row_split = imm_clus, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38","#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha, right_annotation = ha2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-5-1.png)

``` r
p1 <- Heatmap(cellprop_rnaseq, row_split = imm_clus, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38","#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha)


#pdf('~/Dog/TME_manuscript1.pdf')
#Heatmap(cellprop_rnaseq, row_split = imm_clus, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha)
#dev.off()

names(imm_clus) <- rownames(cellprop_rnaseq)
names(ii1) <- rownames(cellprop_rnaseq)

cyt_scores <- CYT_signature(canin_exp)
names(cyt_scores) <- rownames(cellprop_rnaseq)


cell_props_dat <- as.data.frame(cellprop_rnaseq)
cell_props_dat$imm_clus <- imm_clus
Heatmap(cellprop_rnaseq, row_split = imm_clus, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38","#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha, right_annotation = ha2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-5-2.png)

``` r
#pdf('~/Dog/MCP_marker_genes_expression.pdf',width = 18,height = 9)
#Heatmap(cellprop_rnaseq, row_split = imm_clus, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38","#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha, right_annotation = ha2)
#dev.off()

p1 <- ggplot(data = data.frame(subtype = imm_clus, CIN_score = CIN_signature(log2(canin_exp+1))), aes(x = subtype, y=CIN_score)) + geom_boxplot() + theme_bw() + theme(text = element_text(size = 15), axis.text.x = element_text(size = 15)) + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.signif")

#+ stat_compare_means(method = "anova", label = "p.format")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-5-3.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/CIN_subtypes.pdf", width = 7, height = 6)
p2 <- ggplot(data = data.frame(subtype = imm_clus, CGAS_STING_score = cGAS_STING_signature(canin_exp)), aes(x = subtype, y=CGAS_STING_score)) + geom_boxplot() + theme_bw() + theme(text = element_text(size = 15), axis.text.x = element_text(size = 15)) + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.signif")
print(p2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-5-4.png)

``` r
p2 <- ggplot(data = data.frame(subtype = imm_clus, IN_score = type1_interferon_signature(log2(canin_exp+1))), aes(x = subtype,y=IN_score)) + geom_boxplot() +theme_bw() + stat_compare_means(method = "anova")
print(p2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-5-5.png)

``` r
#ggsave(plot = p2, filename = "~/Dog/IN_caines.pdf", width = 4, height = 4)


p2 <- ggplot(data = data.frame(subtype = imm_clus, CYT_score = cyt_scores), aes(x = subtype,y=CYT_score)) + geom_boxplot() +theme_bw() + stat_compare_means(method = "anova") + ylab("Cytotoxicity scores")
print(p2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-5-6.png)

``` r
#ggsave(plot = p2, filename = "~/Dog/CYT_caines.pdf", width = 4, height = 4)
```

# Assessing reproducibility of TME clusters via MCP counter analysis using canine single cell RNA seq cell type signatures (Ammons et al, Communications Biology, 2024)

``` r
library(ConsensusClusterPlus)
set.seed(2425)
Ammons_signatures <- read_excel("~/Dog/Ammons_signatures.xlsx")
genes <- c()
cell_types <- c()
cell_subtypes <- c()
for(i in 1:nrow(Ammons_signatures)){
  genes <- c(genes, strsplit(Ammons_signatures$Markers[i],split = ", ")[[1]])
  cell_types <- c(cell_types, rep(Ammons_signatures$`Cell type`[i], length(strsplit(Ammons_signatures$Markers[i],split = ", ")[[1]])))
  cell_subtypes <- c(cell_subtypes, rep(Ammons_signatures$`Cell type detailed`[i], length(strsplit(Ammons_signatures$Markers[i],split = ", ")[[1]])))
}

markers_ammons <- data.frame(HUGO_symbols = genes, Cell_population = cell_subtypes)
colnames(markers_ammons) <- c("HUGO symbols","Cell population")

cellprop_rnaseq2<- t(MCPcounter.estimate(expression = log2(canin_exp+1), featuresType = "HUGO_symbols", genes = markers_ammons))
cellprop_rnaseq2 <- apply(cellprop_rnaseq2, 2, scale)
rownames(cellprop_rnaseq2) <- colnames(canin_exp)

res = ConsensusClusterPlus(d = t(cellprop_rnaseq2), distance = "euclidean", clusterAlg = "km", pItem = 0.8, pFeature = 1, seed = 2425,maxK = 10, reps = 100,plot = "pdf",title = "ConsensusAmmonsRNASeq")
```

    ## end fraction

    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered

``` r
ha = rowAnnotation(
    sample = sample_type,
    col = list("sample" = c("Primary" = "yellow", "Met" =  "darkgreen")
    )
)

imm_clus_ammons <- rep("ID", nrow(cellprop_rnaseq2))
imm_clus_ammons[as.character(res[[3]]$consensusClass) == "3"] <- "IE"
imm_clus_ammons[as.character(res[[3]]$consensusClass) == "2"] <- "IE-ECM"
names(imm_clus_ammons) <- rownames(cellprop_rnaseq2)
Heatmap(cellprop_rnaseq2, row_split = factor(imm_clus_ammons, levels = c("IE","IE-ECM","ID")), cluster_columns = T, cluster_row_slices = FALSE, row_gap = unit(3, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-6-1.png)

``` r
rownames(cellprop_rnaseq2) <- NULL
pdf('~/Dog/TME_manuscript_ammons.pdf', width = "13",height = "11")
```

    ## Warning: 'mode(width)' and 'mode(height)' differ between new and previous
    ##   ==> NOT changing 'width' & 'height'

``` r
Heatmap(cellprop_rnaseq2, row_split = factor(imm_clus_ammons, levels = c("IE","IE-ECM","ID")), cluster_columns = T, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha)
dev.off()
```

    ## quartz_off_screen 
    ##                 2

``` r
chisq.test(table(factor(imm_clus,levels = c("IE","IE-ECM","ID")), factor(imm_clus_ammons, levels = c("IE","IE-ECM","ID"))))
```

    ## Warning in chisq.test(table(factor(imm_clus, levels = c("IE", "IE-ECM", :
    ## Chi-squared approximation may be incorrect

    ## 
    ##  Pearson's Chi-squared test
    ## 
    ## data:  table(factor(imm_clus, levels = c("IE", "IE-ECM", "ID")), factor(imm_clus_ammons,     levels = c("IE", "IE-ECM", "ID")))
    ## X-squared = 105.4, df = 4, p-value < 2.2e-16

# Validation of TME subtypes through quantitative imaging analysis usig matched H&E and IHC slides

``` r
library(readr)
library(readxl)
library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
trichrome_quant_updated <- read_csv("~/Dog/trichrome_MLH_quant_updated4.csv")
```

    ## Rows: 113 Columns: 17

    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (7): Image, Object ID, Object type, Classification, Parent, ROI, ROI ann...
    ## dbl (9): Centroid X µm, Centroid Y µm, trichrome_threholder_hires: Tumor %, ...
    ## lgl (1): Name
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
osteoid_quant_updated <- read_csv("~/Dog/osteoid_quant_MLH.csv")
```

    ## Rows: 113 Columns: 10
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (6): Image Tag, Algorithm Name, Algorithm Name.1, Analysis Region, Analy...
    ## dbl (4): QC Slide V2: % Artifact Area, Tumor: % Osteoid, Tumor: % Osteoid.1,...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
#Filtering out poor quality cases post manual review
osteoid_quant_updated <- osteoid_quant_updated[!(osteoid_quant_updated$`Image Tag` %in% c("2023-09-16-03-34-18.ndpi",
                                                                                          "2023-09-20-13-12-21.ndpi",
                                                                                          "2023-09-23-17-29-02.ndpi",
                                                                                          "2023-09-26-23-32-10.ndpi",
                                                                                          "2023-09-27-12-06-34.ndpi",
                                                                                          "2023-09-27-17-26-47.ndpi")),]

tcell_quant_updated <- read_csv("~/Dog/tcell_quant_MLH.csv")
```

    ## Rows: 112 Columns: 16
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr  (6): Image, Name, Class, Parent, ROI, ROI annotations completed
    ## dbl (10): Centroid X µm, Centroid Y µm, Num Detections, Num Negative, Num Po...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
#Filtering out poor quality cases post manual review
tcell_quant_updated <- tcell_quant_updated[!(grepl('bad section', tcell_quant_updated$`ROI annotations completed`)),]
macrophage_quant_updated = read_csv("~/Dog/macrophage_quant_MLH.csv")
```

    ## Rows: 111 Columns: 16
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr  (6): Image, Name, Class, Parent, ROI, ROI annotations completed
    ## dbl (10): Centroid X µm, Centroid Y µm, Num Detections, Num Negative, Num Po...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
#Filtering out poor quality cases post manual review
macrophage_quant_updated <- macrophage_quant_updated[!(grepl('bad section', macrophage_quant_updated$`ROI annotations completed`)) & !(macrophage_quant_updated$Image == "2023-09-22-11-22-33.ndpi"),]


mac_density = data.frame(id = macrophage_quant_updated$`Subject Code`, density = macrophage_quant_updated$`Positive %`)
mac_density = mac_density %>% group_by(id) %>% summarize(density = mean(density, na.rm=TRUE))

tcell_density = data.frame(id = tcell_quant_updated$`Subject Code`, density = tcell_quant_updated$`Positive %`)
tcell_density = tcell_density %>% group_by(id) %>% summarize(density = mean(density, na.rm=TRUE))

tri_density = data.frame(id = trichrome_quant_updated$`Subject Code`, area = trichrome_quant_updated$`trichrome_threholder_hires: Collagen %`)
tri_density = tri_density %>% group_by(id) %>% summarize(area = mean(area, na.rm=TRUE))

osteoid_density = data.frame(id = osteoid_quant_updated$`Subject Code`, area = osteoid_quant_updated$`Tumor: % Osteoid`)
osteoid_density = osteoid_density %>% group_by(id) %>% summarize(area = mean(area, na.rm=TRUE))


tcell_density$subtype = ii1[as.character(tcell_density$id)]
tcell_density <- tcell_density[!is.na(tcell_density$subtype),]
tcell_density$subtype <- factor(tcell_density$subtype, levels = c("ID","IE-ECM","IE"))
mac_density$subtype = ii1[as.character(mac_density$id)]
mac_density <- mac_density[!is.na(mac_density$subtype),]
mac_density$subtype <- factor(mac_density$subtype, levels = c("ID","IE-ECM","IE"))
tri_density$subtype = ii1[as.character(tri_density$id)]
tri_density <- tri_density[!is.na(tri_density$subtype),]
tri_density$subtype <- factor(tri_density$subtype, levels = c("ID","IE-ECM","IE"))
osteoid_density$subtype = ii1[as.character(osteoid_density$id)]
osteoid_density <- osteoid_density[!is.na(osteoid_density$subtype),]
osteoid_density$subtype <- factor(osteoid_density$subtype, levels = c("ID","IE-ECM","IE"))

tcell_density$exp <- cellprop_rnaseq[as.character(tcell_density$id),"T cells"]
mac_density$exp <- cellprop_rnaseq[as.character(mac_density$id),"Monocytic lineage"]
tri_density$exp <- cellprop_rnaseq[as.character(tri_density$id),"Fibroblasts"]
osteoid_density$exp <- cellprop_rnaseq[as.character(osteoid_density$id),"Fibroblasts"]

p1 <- ggplot(data = tcell_density, aes(x=subtype, y = density)) + geom_boxplot() + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.format") + theme_minimal() + ylab("%Positive") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "CD3 IHC (T cells)")


p2 <- ggplot(data = mac_density, aes(x=subtype, y = density)) + geom_boxplot() + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.format") + theme_minimal() + ylab("%Positive") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "CD204 IHC (Macrophage)")


p3 <- ggplot(data = tri_density, aes(x=subtype, y = area)) + geom_boxplot() + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.format") + theme_minimal() + ylab("%Collagen") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "Trichrome (Collagen)")

p33 <- ggplot(data = osteoid_density, aes(x=subtype, y = area)) + geom_boxplot() + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.format") + theme_minimal() + ylab("%Osteoid") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "Osteoid")

p6 <- ggplot(data = tri_density, aes(x = exp, y= area)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 1,
            label = paste0('r = ', round(cor(tri_density$area, tri_density$exp, method = c("pearson")), 4)),
            color = 'red') + ylab("%Collagen") + xlab("MCP counter score (Stroma)") + theme(text = element_text(size = 15)) + labs(title = "Trichrome (Collagen)")

p5 <- ggplot(data = mac_density, aes(x = exp, y= density)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 1,
            label = paste0('r = ', round(cor(mac_density$density, mac_density$exp, method = c("pearson")), 4)),
            color = 'red') + ylab("%Positive") + xlab("MCP counter score (Monocytic lineage)") + theme(text = element_text(size = 15)) + labs(title = "CD204 IHC (Macrophage)")

p4 <- ggplot(data = tcell_density, aes(x = exp, y= density)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 1,
            label = paste0('r = ', round(cor(tcell_density$density, tcell_density$exp, method = c("pearson")), 4)),
            color = 'red') + ylab("%Positive") + xlab("MCP counter score (T cells)") + theme(text = element_text(size = 15)) + labs(title = "CD3 IHC (T cells)")

p66 <- ggplot(data = osteoid_density, aes(x = exp, y= area)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 1,
            label = paste0('r = ', round(cor(osteoid_density$area, osteoid_density$exp, method = c("pearson")), 4)),
            color = 'red') + ylab("%Osteoid") + xlab("MCP counter score (Stroma)") + theme(text = element_text(size = 15)) + labs(title = "Osteoid")

fig1 <- ggarrange(p1,p2,p3,p33,p4,p5,p6,p66, nrow = 2, ncol = 4, labels = c("A","B","C","D","E","F","G","H"))
```

    ## `geom_smooth()` using formula = 'y ~ x'
    ## `geom_smooth()` using formula = 'y ~ x'
    ## `geom_smooth()` using formula = 'y ~ x'
    ## `geom_smooth()` using formula = 'y ~ x'

``` r
print(fig1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-7-1.png)

``` r
########################################################################################################################

mac_density = data.frame(id = macrophage_quant_updated$`Subject Code`, density = macrophage_quant_updated$`Num Positive per mm^2`)
mac_density = mac_density %>% group_by(id) %>% summarize(density = mean(density, na.rm=TRUE))

tcell_density = data.frame(id = tcell_quant_updated$`Subject Code`, density = tcell_quant_updated$`Num Positive per mm^2`)
tcell_density = tcell_density %>% group_by(id) %>% summarize(density = mean(density, na.rm=TRUE))

tri_density = data.frame(id = trichrome_quant_updated$`Subject Code`, area = trichrome_quant_updated$`trichrome_threholder_hires: Collagen %`)
tri_density = tri_density %>% group_by(id) %>% summarize(area = mean(area, na.rm=TRUE))

osteoid_density = data.frame(id = osteoid_quant_updated$`Subject Code`, area = osteoid_quant_updated$`Tumor: % Osteoid`)
osteoid_density = osteoid_density %>% group_by(id) %>% summarize(area = mean(area, na.rm=TRUE))



tcell_density$subtype = ii1[as.character(tcell_density$id)]
tcell_density <- tcell_density[!is.na(tcell_density$subtype),]
tcell_density$subtype <- factor(tcell_density$subtype, levels = c("ID","IE-ECM","IE"))
mac_density$subtype = ii1[as.character(mac_density$id)]
mac_density <- mac_density[!is.na(mac_density$subtype),]
mac_density$subtype <- factor(mac_density$subtype, levels = c("ID","IE-ECM","IE"))
tri_density$subtype = ii1[as.character(tri_density$id)]
tri_density <- tri_density[!is.na(tri_density$subtype),]
tri_density$subtype <- factor(tri_density$subtype, levels = c("ID","IE-ECM","IE"))
osteoid_density$subtype = ii1[as.character(osteoid_density$id)]
osteoid_density <- osteoid_density[!is.na(osteoid_density$subtype),]
osteoid_density$subtype <- factor(osteoid_density$subtype, levels = c("ID","IE-ECM","IE"))

tcell_density$exp <- cellprop_rnaseq[as.character(tcell_density$id),"T cells"]
mac_density$exp <- cellprop_rnaseq[as.character(mac_density$id),"Monocytic lineage"]
tri_density$exp <- cellprop_rnaseq[as.character(tri_density$id),"Fibroblasts"]
osteoid_density$exp <- cellprop_rnaseq[as.character(osteoid_density$id),"Fibroblasts"]


p1 <- ggplot(data = tcell_density, aes(x=subtype, y = density)) + geom_boxplot() + stat_compare_means(method = "anova") + theme_minimal() + ylab("Density") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "CD3 IHC (T cells)")


p2 <- ggplot(data = mac_density, aes(x=subtype, y = density)) + geom_boxplot() + stat_compare_means(method = "anova") + theme_minimal() + ylab("Density") + xlab("Suntype") + theme(text = element_text(size = 15)) + labs(title = "CD204 IHC (Macrophage)")


p3 <- ggplot(data = tri_density, aes(x=subtype, y = area)) + geom_boxplot() + stat_compare_means(method = "anova") + theme_minimal() + ylab("%Collagen") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "Trichrome (Collagen)")

p33 <- ggplot(data = osteoid_density, aes(x=subtype, y = area)) + geom_boxplot() + stat_compare_means(method = "anova") + theme_minimal() + ylab("%Osteoid") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "Osteoid")

p6 <- ggplot(data = tri_density, aes(x = exp, y= area)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 10,
            label = paste0('Spearman rho = ', round(cor(tri_density$area, tri_density$exp, method = c("spearman")), 4)),
            color = 'red') + ylab("%Collagen") + xlab("MCP counter score (Stroma)") + theme(text = element_text(size = 15)) + labs(title = "Trichrome (Collagen)")

p5 <- ggplot(data = mac_density, aes(x = exp, y= density)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 1,
            label = paste0('Spearman rho = ', round(cor(mac_density$density, mac_density$exp, method = c("spearman")), 4)),
            color = 'red') + ylab("Density") + xlab("MCP counter score (Monocytic lineage)") + theme(text = element_text(size = 15)) + labs(title = "CD204 IHC (Macrophage)")

p4 <- ggplot(data = tcell_density, aes(x = exp, y= density)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 1,
            label = paste0('Spearman rho = ', round(cor(tcell_density$density, tcell_density$exp, method = c("spearman")), 4)),
            color = 'red') + ylab("Density") + xlab("MCP counter score (T cells)") + theme(text = element_text(size = 15)) + labs(title = "CD3 IHC (T cells)")

p66 <- ggplot(data = osteoid_density, aes(x = exp, y= area)) + geom_point()+ geom_smooth(method = "lm", se = FALSE) + theme_minimal() + geom_text(x = 1, y = 1,
            label = paste0('Spearman rho = ', round(cor(osteoid_density$area, osteoid_density$exp, method = c("spearman")), 4)),
            color = 'red') + ylab("%Osteoid") + xlab("MCP counter score (Stroma)") + theme(text = element_text(size = 15)) + labs(title = "Osteoid")

fig2 <- ggarrange(p1,p2,p3,p33,p4,p5,p6,p66, nrow = 2, ncol = 4, labels = c("A","B","C","D","E","F","G","H"))
```

    ## `geom_smooth()` using formula = 'y ~ x'
    ## `geom_smooth()` using formula = 'y ~ x'
    ## `geom_smooth()` using formula = 'y ~ x'
    ## `geom_smooth()` using formula = 'y ~ x'

``` r
print(fig2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-7-2.png)

``` r
p1 <- ggplot(data = tcell_density, aes(x=subtype, y = density)) + geom_boxplot()  + stat_compare_means(method = "anova") + theme_minimal() + ylab("Density") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "CD3 IHC (T cells)") + scale_y_continuous(limits = c(0,4500))


p2 <- ggplot(data = mac_density[mac_density$density < 6000,], aes(x=subtype, y = density)) + geom_boxplot()  + stat_compare_means(method = "anova") + theme_minimal() + ylab("Density") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "CD204 IHC (Macrophage)") + scale_y_continuous(limits = c(0,4500))


p3 <- ggplot(data = tri_density, aes(x=subtype, y = area)) + geom_boxplot()  + stat_compare_means(method = "anova") + theme_minimal() + ylab("%Collagen") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "Trichrome (Collagen)")

p33 <- ggplot(data = osteoid_density, aes(x=subtype, y = area)) +  geom_boxplot()  + stat_compare_means(method = "anova") + theme_minimal() + ylab("%Osteoid") + xlab("Subtype") + theme(text = element_text(size = 15)) + labs(title = "Osteoid")

fig3 <- ggarrange(p1,p2,p3,p33, nrow = 2, ncol = 2, labels = c("A","B","C","D"))
print(fig3)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-7-3.png)

``` r
#ggsave(plot = fig3, filename = "~/Dog/Final_IHC_validation_figure_panels.pdf", width = 6, height = 6)
```

# Plotting distribution of immune checkpoint gene expression across TME clusters

``` r
library(reshape)
```

    ## 
    ## Attaching package: 'reshape'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     rename

``` r
out <- t(log2(canin_exp[intersect(c("PDCD1","CD274","CTLA4","TIGIT","LAG3","HAVCR2"), rownames(canin_exp)),]+1))
out <- data.frame(apply(out,2,scale), subtype = imm_clus)

dd_out <- melt(out, id.vars = "subtype")
colnames(dd_out) <- c("subtype","ICI_target","expression")
dd_out$ICI_target <- as.character(dd_out$ICI_target)
dd_out$subtype <- factor(dd_out$subtype, levels = c("IE","IE-ECM","ID"))
dd_out$ICI_target[dd_out$ICI_target == "PDCD1"] = "PD1"
dd_out$ICI_target[dd_out$ICI_target == "CD274"] = "PDL1"

#plot results
p1 <- ggboxplot(dd_out, x = "subtype", y="expression", facet.by = "ICI_target") + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means(method = "anova")+ xlab("") + ylab("expression (normalized)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-8-1.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/canine_immune_checkpoints.pdf")
```

# Assessing reproducibility of TME clusters via xCell analysis

``` r
library(xCell)
out <- t(xCellAnalysis(canin_exp))
```

    ## [1] "Num. of genes: 9787"

    ## Warning: useNames = NA is deprecated. Instead, specify either useNames = TRUE
    ## or useNames = FALSE.

    ## Setting parallel calculations through a MulticoreParam back-end
    ## with workers=4 and tasks=100.
    ## Estimating ssGSEA scores for 489 gene sets.
    ## [1] "Calculating ranks..."

    ## Warning: useNames = NA is deprecated. Instead, specify either useNames = TRUE
    ## or useNames = FALSE.

    ## [1] "Calculating absolute values from ranks..."
    ##   |                                                                              |                                                                      |   0%  |                                                                              |=                                                                     |   1%  |                                                                              |=                                                                     |   2%  |                                                                              |==                                                                    |   3%  |                                                                              |===                                                                   |   4%  |                                                                              |====                                                                  |   5%  |                                                                              |====                                                                  |   6%  |                                                                              |=====                                                                 |   7%  |                                                                              |======                                                                |   8%  |                                                                              |======                                                                |   9%  |                                                                              |=======                                                               |  10%  |                                                                              |========                                                              |  11%  |                                                                              |========                                                              |  12%  |                                                                              |=========                                                             |  13%  |                                                                              |==========                                                            |  14%  |                                                                              |===========                                                           |  15%  |                                                                              |===========                                                           |  16%  |                                                                              |============                                                          |  17%  |                                                                              |=============                                                         |  18%  |                                                                              |=============                                                         |  19%  |                                                                              |==============                                                        |  20%  |                                                                              |===============                                                       |  21%  |                                                                              |================                                                      |  22%  |                                                                              |================                                                      |  23%  |                                                                              |=================                                                     |  24%  |                                                                              |==================                                                    |  25%  |                                                                              |==================                                                    |  26%  |                                                                              |===================                                                   |  27%  |                                                                              |====================                                                  |  28%  |                                                                              |=====================                                                 |  29%  |                                                                              |=====================                                                 |  30%  |                                                                              |======================                                                |  31%  |                                                                              |=======================                                               |  32%  |                                                                              |=======================                                               |  33%  |                                                                              |========================                                              |  34%  |                                                                              |=========================                                             |  35%  |                                                                              |=========================                                             |  36%  |                                                                              |==========================                                            |  37%  |                                                                              |===========================                                           |  38%  |                                                                              |============================                                          |  39%  |                                                                              |============================                                          |  40%  |                                                                              |=============================                                         |  41%  |                                                                              |==============================                                        |  42%  |                                                                              |==============================                                        |  43%  |                                                                              |===============================                                       |  44%  |                                                                              |================================                                      |  45%  |                                                                              |=================================                                     |  46%  |                                                                              |=================================                                     |  47%  |                                                                              |==================================                                    |  48%  |                                                                              |===================================                                   |  49%  |                                                                              |===================================                                   |  51%  |                                                                              |====================================                                  |  52%  |                                                                              |=====================================                                 |  53%  |                                                                              |=====================================                                 |  54%  |                                                                              |======================================                                |  55%  |                                                                              |=======================================                               |  56%  |                                                                              |========================================                              |  57%  |                                                                              |========================================                              |  58%  |                                                                              |=========================================                             |  59%  |                                                                              |==========================================                            |  60%  |                                                                              |==========================================                            |  61%  |                                                                              |===========================================                           |  62%  |                                                                              |============================================                          |  63%  |                                                                              |=============================================                         |  64%  |                                                                              |=============================================                         |  65%  |                                                                              |==============================================                        |  66%  |                                                                              |===============================================                       |  67%  |                                                                              |===============================================                       |  68%  |                                                                              |================================================                      |  69%  |                                                                              |=================================================                     |  70%  |                                                                              |=================================================                     |  71%  |                                                                              |==================================================                    |  72%  |                                                                              |===================================================                   |  73%  |                                                                              |====================================================                  |  74%  |                                                                              |====================================================                  |  75%  |                                                                              |=====================================================                 |  76%  |                                                                              |======================================================                |  77%  |                                                                              |======================================================                |  78%  |                                                                              |=======================================================               |  79%  |                                                                              |========================================================              |  80%  |                                                                              |=========================================================             |  81%  |                                                                              |=========================================================             |  82%  |                                                                              |==========================================================            |  83%  |                                                                              |===========================================================           |  84%  |                                                                              |===========================================================           |  85%  |                                                                              |============================================================          |  86%  |                                                                              |=============================================================         |  87%  |                                                                              |==============================================================        |  88%  |                                                                              |==============================================================        |  89%  |                                                                              |===============================================================       |  90%  |                                                                              |================================================================      |  91%  |                                                                              |================================================================      |  92%  |                                                                              |=================================================================     |  93%  |                                                                              |==================================================================    |  94%  |                                                                              |==================================================================    |  95%  |                                                                              |===================================================================   |  96%  |                                                                              |====================================================================  |  97%  |                                                                              |===================================================================== |  98%  |                                                                              |===================================================================== |  99%  |                                                                              |======================================================================| 100%

``` r
res = ConsensusClusterPlus(d = t(apply(out,2,scale)), distance = "euclidean", clusterAlg = "km", pItem = 0.8, pFeature = 1, seed = 2425,maxK = 10, reps = 100, plot = "pdf",title = "ConsensusXCellRNASeq")
```

    ## end fraction

    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered
    ## clustered

``` r
ha = rowAnnotation(
    sample = sample_type,
    col = list("sample" = c("Primary" = "yellow", "Met" =  "darkgreen")
    )
)

#plot results
Heatmap(apply(out,2,scale), row_split = as.character(res[[3]]$consensusClass), cluster_columns = T, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha)
```

    ## The automatically generated colors map from the minus and plus 99^th of
    ## the absolute values in the matrix. There are outliers in the matrix
    ## whose patterns might be hidden by this color mapping. You can manually
    ## set the color to `col` argument.
    ## 
    ## Use `suppressMessages()` to turn off this message.

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-9-1.png)

``` r
#pdf('~/Dog/TME_manuscript3.pdf', width = "13",height = "11")
#Heatmap(apply(out,2,scale), row_split = imm_clus, cluster_columns = T, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")), row_names_gp = gpar(cex = 0.5), left_annotation = ha)
#dev.off()
```

# run MCP counter + clustering analysis on bulk nanostring data

``` r
set.seed(24524)
library(RColorBrewer)
cellprop_nano <- t(MCPcounter.estimate(expression = log2(cbind(CE[,NaAn$`Sample Type` %in% c("Primary Tumor")],CE[,NaAn$`Sample Type` %in% c("Met")])+1), featuresType = "HUGO_symbols", probesets = read.table("~/Dog/MCP_counter_markers.csv")))

#expression of immune checkpoint genes in nanostring
chkpt_nano <- cbind(CE[,NaAn$`Sample Type` %in% c("Primary Tumor")],CE[,NaAn$`Sample Type` %in% c("Met")])
chkpt_nano <- t(chkpt_nano[intersect(rownames(chkpt_nano), c("PDCD1","CD274","CTLA4","LAG3","HAVCR2","TIGIT")),])


nanostring_annotations <- rbind(NaAn[NaAn$`Sample Type` %in% c("Primary Tumor"),], NaAn[NaAn$`Sample Type` %in% c("Met"),])

sample_preservation = c("Frozen","FFPE")[as.numeric(grepl("FFPE",nanostring_annotations$`Name, detailed`))+1]

#perform sample preservation-specific normalization to mitigate batch effects across different sample preservation techniques
cellprop_nano_ffpe <- apply(cellprop_nano[sample_preservation == "FFPE",],2,scale)
chkpt_nano_ffpe <- apply(chkpt_nano[sample_preservation == "FFPE",],2,scale)
km.res <- kmeans(cellprop_nano_ffpe, centers = 3, nstart = 100, iter.max = 500)

imm_clus_nano_ffpe <- rep("ID", nrow(cellprop_nano_ffpe))
imm_clus_nano_ffpe[km.res$cluster == 2] <- "IE"
imm_clus_nano_ffpe[km.res$cluster == 1] <- "IE-ECM"
Heatmap(cellprop_nano_ffpe, row_split = imm_clus_nano_ffpe, cluster_column_slices = F, cluster_columns = F)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-10-1.png)

``` r
cellprop_nano_frozen <- apply(cellprop_nano[sample_preservation == "Frozen",],2,scale)
chkpt_nano_frozen <- apply(chkpt_nano[sample_preservation == "Frozen",],2,scale)

km.res <- kmeans(cellprop_nano_frozen, centers = 3, nstart = 100, iter.max = 500)

imm_clus_nano_frozen <- rep("ID", nrow(cellprop_nano_frozen))
imm_clus_nano_frozen[km.res$cluster == 3] <- "IE"
imm_clus_nano_frozen[km.res$cluster == 2] <- "IE-ECM"
Heatmap(cellprop_nano_frozen, row_split = imm_clus_nano_frozen, cluster_column_slices = F, cluster_columns = F)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-10-2.png)

``` r
#merge FFPe and frozen sample results
cellprop_nano <- rbind(cellprop_nano_ffpe, cellprop_nano_frozen)
chkpt_nano <- rbind(chkpt_nano_ffpe, chkpt_nano_frozen)
nanostring_annotations <- rbind(nanostring_annotations[sample_preservation == "FFPE",],nanostring_annotations[sample_preservation == "Frozen",])
imm_clus_nano = c(imm_clus_nano_ffpe, imm_clus_nano_frozen)
sample_preservation <- c(sample_preservation[sample_preservation == "FFPE"],sample_preservation[sample_preservation == "Frozen"])


#Primary vs metastatic tumor sample status
sstatus = nanostring_annotations$`Sample Type`
ha = rowAnnotation(
    sample = sstatus,
    col = list("sample" = c("Primary Tumor" = "yellow", "Met" =  "green")
    )
)

#metastatic site information
ss <- toupper(nanostring_annotations$Site)
ss_split <- rep("OTHER",length(ss))
ss_split[ss == "LUNG"] <- "LUNG"
ss_split[ss == "KIDNEY"] <- "KIDNEY"
ss_split[ss == "LIVER"] <- "LIVER"
ss_split[ss %in% c("BONE","RIB","L4","T5","T6","T11/T13","SPINE")] <- "SKELETAL"
ss_split[ss == "PRIMARY"] <- "PRIMARY"
n <- 70
qual_col_pals = brewer.pal.info[brewer.pal.info$category == 'qual',]
col_vector = unlist(mapply(brewer.pal, qual_col_pals$maxcolors, rownames(qual_col_pals)))
cols <- sample(col_vector, n, replace = F)
cols <- sample(cols, length(unique(ss)), replace = F)
names(cols) <- unique(ss)



#plot results
ha = rowAnnotation(
    sample = sstatus,
    preservation = sample_preservation,
    col = list("sample" = c("Primary Tumor" = "yellow", "Met" =  "darkgreen"),
               "preservation" = c("Frozen" = "#db6d00", "FFPE" = "#924900")
    )
)

ha2 = rowAnnotation(
    subtype = imm_clus_nano,
    col = list("subtype" = c("IE" = "#882255", "IE-ECM" =  "#CC6677", "ID" = "#88CCEE"))
)



names(imm_clus_nano) <- as.character(as.numeric(nanostring_annotations$`Subject Code`))
ii <- imm_clus_nano
imm_clus_nano <- factor(imm_clus_nano, levels = c("IE","IE-ECM", "ID"))
rownames(cellprop_nano) <- names(ii)
Heatmap(cellprop_nano, row_split = ss_split, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), left_annotation = ha, right_annotation = ha2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-10-3.png)

``` r
#pdf("~/Dog/metTME_landscape.pdf", height = 14, width = 8)
#Heatmap(cellprop_nano, row_split = ss_split, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(3, #"mm"), left_annotation = ha, right_annotation = ha2)
#dev.off()
#rownames(cellprop_nano) <- samp_old
```

#Plotting distribution of immune checkpoint gene expression across TME
clusters (Nanostring)

``` r
Heatmap(chkpt_nano, row_split = ss_split, cluster_columns = T, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), left_annotation = ha, right_annotation = ha2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-11-1.png)

``` r
out <- data.frame(chkpt_nano, subtype = imm_clus_nano)
dd_out <- melt(out, id.vars = "subtype")
colnames(dd_out) <- c("subtype","ICI_target","scaled_expression")
dd_out$ICI_target <- as.character(dd_out$ICI_target)
dd_out$subtype <- factor(dd_out$subtype, levels = c("IE","IE-ECM","ID"))
dd_out$ICI_target[dd_out$ICI_target == "PDCD1"] = "PD1"
dd_out$ICI_target[dd_out$ICI_target == "CD274"] = "PDL1"

p1 <- ggboxplot(dd_out, x = "subtype", y="scaled_expression", facet.by = "ICI_target") + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means(method = "anova") + xlab("") + ylab("expression (normalized)")
#stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.signif")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-11-2.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/canine_immune_checkpoints_nano.pdf")
```

# Joint analysis of primary and metastatic tumor samples from dogs enolled in COTC021/22

``` r
library(tibble)
set.seed(317796929)

#All metastatic samples (Nanostring + RNASeq)
a1 <- rbind(cellprop_nano[sstatus == "Met",], cellprop_rnaseq[sample_type == "Met",])
source_a1 <- c(rep("Nanostring", nrow(cellprop_nano[sstatus == "Met",])), rep("RNASeq",nrow(cellprop_rnaseq[sample_type == "Met",])))
prep_a1 <- c(sample_preservation[sstatus == "Met"], rep("Frozen",nrow(cellprop_rnaseq[sample_type == "Met",])))
imm_clus_a1 <- c(ii[sstatus == "Met"], ii1[sample_type == "Met"])
sites_a1 <- c(ss[sstatus == "Met"], sites_RNASeq[sample_type == "Met"])

#All primary samples (Nanostring + RNASeq)
a2 <- rbind(cellprop_rnaseq[sample_type == "Primary",], cellprop_nano[sstatus == "Primary Tumor",])
source_a2 <- c(rep("RNASeq", nrow(cellprop_rnaseq[sample_type == "Primary",])), rep("Nanostring",nrow(cellprop_nano[sstatus == "Primary Tumor",])))
prep_a2 <- c(rep("Frozen",nrow(cellprop_rnaseq[sample_type == "Primary",])), sample_preservation[sstatus == "Primary Tumor"])
imm_clus_a2 <- c(ii1[sample_type == "Primary"], ii[sstatus == "Primary Tumor"])
sites_a2 <- rep("Primary", length(imm_clus_a2))

#match tumor samples from same patient
common <- intersect(rownames(a1), rownames(a2))
case_list <- list()
for(pt in common){
    xx <- a1[rownames(a1) == pt,]
    if(is.null(dim(xx))){
        xx <- t(xx)
    }
    d1 <- data.frame(xx)
    d1$sample_type = "Metastatic"
    d1$source = source_a1[rownames(a1) == pt]
    d1$prep = prep_a1[rownames(a1) == pt]
    d1$site = sites_a1[rownames(a1) == pt]
    d1$case = pt
    d1$immune_subtype = imm_clus_a1[rownames(a1) == pt]
    
    xx <- a2[rownames(a2) == pt,]
    if(is.null(dim(xx))){
        xx <- t(xx)
    }
    d2 <- data.frame(xx)
    d2$sample_type = "Primary"
    d2$source = source_a2[rownames(a2) == pt]
    d2$prep = prep_a2[rownames(a2) == pt]
    d2$site = sites_a2[rownames(a2) == pt]
    d2$case = pt
    d2$immune_subtype = imm_clus_a2[rownames(a2) == pt]
    
    case_list[[pt]] <- rbind(d2,d1)
}
matched_cases <- Reduce(rbind, case_list)

matched_cases$sample_type <- factor(matched_cases$sample_type, levels = c("Primary", "Metastatic"))
ss1_split <- rep("OTHER",nrow(matched_cases))
ss1_split[matched_cases$site == "LUNG"] <- "LUNG"
ss1_split[matched_cases$site == "KIDNEY"] <- "KIDNEY"
ss1_split[matched_cases$site == "LIVER"] <- "LIVER"
ss1_split[matched_cases$site %in% c("BONE","RIB","L4","T5","T6","T11/T13","SPINE")] <- "SKELETAL"
ss1_split[matched_cases$site == "Primary"] <- "PRIMARY"


#plot results
ha = columnAnnotation(
    site = ss1_split,
    preservation = matched_cases$prep,
    platform = matched_cases$source,
    subtype = matched_cases$immune_subtype,
    col = list("site" = c("OTHER" = "#009E73", 
                          "PRIMARY" =  "#F0E442",
                          "LUNG"="#DF65B0", 
                          "KIDNEY" = "#984EA3",
                          "SKELETAL" = "#FEE5D9",
                          "LIVER" = "#FB6A4A"),
               "preservation" = c("Frozen" = "#db6d00", 
                                  "FFPE" = "#924900"),
               "platform" = c("Nanostring" = "#525252",
                              "RNASeq" = "#F2F0F7"),
               "subtype" = c("ID" = "#88CCEE",
                             "IE" = "#882255",
                             "IE-ECM" = "#CC6677")
    )
)

matched_cases$site2 <- ss1_split

met_mat <- as.matrix(matched_cases[,1:10])
rownames(met_mat) <- matched_cases$case

Heatmap(t(met_mat), column_split =  matched_cases$sample_type, cluster_column_slices = F, column_gap = unit(5, "mm"),  cluster_rows = F, cluster_columns = T, top_annotation = ha, column_names_gp = grid::gpar(fontsize = 8))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-12-1.png)

``` r
sampIds1 <- rownames(a1)
idx <- sapply(sampIds1, function(x) x %in% rownames(a2))
a1 = a1[idx,]
imm_clus_a1 = imm_clus_a1[idx]
ss1 <- sites_a1[idx]

ha = rowAnnotation(
  sample = rep("Primary Tumor", nrow(a2[sapply(rownames(a2), function(x) x %in% sampIds1[idx]),])),
  col = list("Primary Tumor" = "yellow", "Met "= "darkgreen")
)

#pdf("~/Dog/Met_evolve_primary.pdf", width = 5, height = 10)
#Heatmap(a2[sapply(rownames(a2), function(x) x %in% rownames(a1)),], row_split = imm_clus_a2[sapply(rownames(a2), function(x) x %in% rownames(a1))], cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), cluster_rows = F, left_annotation = ha)
#dev.off()



n <- length(unique(ss1))
#qual_col_pals = brewer.pal.info[brewer.pal.info$category == 'qual',]
qual_col_pals = brewer.pal.info[brewer.pal.info$category == 'qual',]
col_vector = unlist(mapply(brewer.pal, qual_col_pals$maxcolors, rownames(qual_col_pals)))
#col_vector = grDevices::colors()[grep('gr(a|e)y', grDevices::colors(), invert = T)]
cols <- sample(col_vector, n, replace = F)
ss1_split <- rep("OTHER",length(ss1))
ss1_split[ss1 == "LUNG"] <- "LUNG"
ss1_split[ss1 == "KIDNEY"] <- "KIDNEY"
ss1_split[ss1 == "LIVER"] <- "LIVER"
ss1_split[ss1 %in% c("BONE","RIB","L4","T5","T6","T11/T13","SPINE")] <- "SKELETAL"
ss1_split[ss1 == "PRIMARY"] <- "PRIMARY"
names(cols) <- unique(ss1)



#samp <- sapply(rownames(a1), function(x) as.character(as.numeric(strsplit(x,"_")[[1]][2])))
#rownames(a1) <- samp
#names(imm_clus_a1) <- rownames(a1)
ha = rowAnnotation(
    met_subtype = imm_clus_a1[imm_clus_a2[names(imm_clus_a1)] == "IE-ECM"],
    col = list("met_subtype" = c("IE" = "#882255", "IE-ECM" =  "#CC6677", "ID" = "#88CCEE")
          )
)

ha2 = rowAnnotation(
  sample = rep("Met", length(imm_clus_a1[imm_clus_a2[names(imm_clus_a1)] == "IE-ECM"])),
  col = list("Primary Tumor"="yellow","Met"="darkgreen")
)

#pdf("~/Dog/Met_evolve_IE-ECM.pdf", width = 5, height = 10)
#Heatmap(a1[imm_clus_a2[names(imm_clus_a1)] == "IE-ECM",], row_split = ss1_split[imm_clus_a2[names(imm_clus_a1)] == "IE-ECM"], cluster_columns = T, cluster_row_slices = FALSE, row_gap = unit(3, "mm"), right_annotation = ha,left_annotation = ha2)
#dev.off()

ha = rowAnnotation(
    met_subtype = imm_clus_a1[imm_clus_a2[names(imm_clus_a1)] == "ID"],
    col = list("met_subtype" = c("IE" = "#882255", "IE-ECM" =  "#CC6677", "ID" = "#88CCEE")
          )
)
ha2 = rowAnnotation(
  sample = rep("Met", length(imm_clus_a1[imm_clus_a2[names(imm_clus_a1)] == "ID"])),
  col = list("Primary Tumor"="yellow","Met"="darkgreen")
)
#pdf("~/Dog/Met_evolve_ID.pdf", width = 5, height = 10)
#Heatmap(a1[imm_clus_a2[names(imm_clus_a1)] == "ID",], row_split = ss1_split[imm_clus_a2[names(imm_clus_a1)] == "ID"], cluster_columns =T, cluster_row_slices = FALSE, row_gap = unit(3, "mm"), right_annotation = ha, left_annotation = ha2)
#dev.off()



#contingency table of matched tumor samples
mat_subtype <- matrix(0,nrow = 3, ncol = 3)
rownames(mat_subtype) <- c("IE","IE-ECM","ID")
colnames(mat_subtype) <- c("IE","IE-ECM","ID")
edge_list <- list()
idx <- 1
for(id in unique(matched_cases$case)){
    case_id <- which(matched_cases$case == id)
    is_prim <- matched_cases$sample_type[case_id] == "Primary"
    is_met <- matched_cases$sample_type[case_id] == "Metastatic"
    for(i in case_id[is_prim]){
        for(j in case_id[is_met]){
            for(subtype_i in c("IE","IE-ECM","ID")){
                for(subtype_j in c("IE","IE-ECM","ID")){
                    if(matched_cases$immune_subtype[i] == subtype_i & matched_cases$immune_subtype[j] == subtype_j){
                        mat_subtype[subtype_i,subtype_j] = mat_subtype[subtype_i, subtype_j] + 1
                        edge_list[[idx]] <- c(i,j)
                        idx <- idx + 1
                    }
                }
            }
        }
    }
}

print(mat_subtype)
```

    ##        IE IE-ECM ID
    ## IE      1      0  0
    ## IE-ECM  7     37 18
    ## ID     12     29 17

``` r
chisq.test(mat_subtype, simulate.p.value = TRUE)
```

    ## 
    ##  Pearson's Chi-squared test with simulated p-value (based on 2000
    ##  replicates)
    ## 
    ## data:  mat_subtype
    ## X-squared = 7.2091, df = NA, p-value = 0.1059

``` r
#generate sankey diagram
library(plotly)
```

    ## 
    ## Attaching package: 'plotly'

    ## The following object is masked from 'package:reshape':
    ## 
    ##     rename

    ## The following object is masked from 'package:ComplexHeatmap':
    ## 
    ##     add_heatmap

    ## The following object is masked from 'package:ggplot2':
    ## 
    ##     last_plot

    ## The following object is masked from 'package:stats':
    ## 
    ##     filter

    ## The following object is masked from 'package:graphics':
    ## 
    ##     layout

``` r
#reticulate::import("sys")
matched_cases$site[matched_cases$site == "SPINAL CORD"] = "EXTRADURAL"
matched_cases$site[matched_cases$site == "LIP"] = "ORAL"

prim_sample <- sapply(edge_list, function(x) paste("primary:",x[1], sep = ""))
met_sample <- sapply(edge_list, function(x) paste(matched_cases$site[x[2]],x[2],sep = ":"))

node_lab <- c(unique(sapply(edge_list, function(x) matched_cases$case[x[1]])), unique(prim_sample), unique(met_sample))

subtype_col <- c("IE" = "#882255", "IE-ECM" =  "#CC6677", "ID" = "#88CCEE")
node_col <- c(rep("grey", length(unique(sapply(edge_list, function(x) matched_cases$case[x[1]])))),
               subtype_col[sapply(unique(prim_sample), function(x) matched_cases$immune_subtype[as.numeric(strsplit(x,split = ":")[[1]][2])])],
               subtype_col[sapply(unique(met_sample), function(x) matched_cases$immune_subtype[as.numeric(strsplit(x,split = ":")[[1]][2])])])

names(node_col) <- NULL

elist1 <- t(sapply(edge_list, function(x) {
  a <- matched_cases$case[x[1]]
  b <- paste("primary:",x[1], sep = "")
  return(c(which(node_lab == a), which(node_lab == b)))
}))

elist2 <- t(sapply(edge_list, function(x) {
  a <- paste("primary:",x[1], sep = "")
  b <- paste(matched_cases$site[x[2]],x[2],sep = ":")
  return(c(which(node_lab == a), which(node_lab == b)))
}))

elist <- rbind(elist1,elist2)
                      

fig <- plot_ly(
    type = "sankey",
    domain = list(
      x =  c(0,1),
      y =  c(0,1)
    ),
    orientation = "h",
    valueformat = ".0f",
    valuesuffix = "TWh",

    node = list(
      label = sapply(node_lab, function(x) strsplit(x,split = ":")[[1]][1]),
      color = node_col,
      pad = 15,
      thickness = 15,
      line = list(
        color = "black",
        width = 0.5
      )
    ),

    link = list(
      source = elist[,1]-1,
      target = elist[,2]-1,
      value = rep(1,nrow(elist))
    )
  ) 

fig <- fig %>% layout(
    title = " ",
    font = list(
      size = 9
    ),
    xaxis = list(showgrid = F, zeroline = F),
    yaxis = list(showgrid = F, zeroline = F)
)

print(fig)
#save_image(fig, file = "/Users/sushantpa/Dog/met_evolve_kaleido.pdf",width = 410, height = 720)
```

# Read clinical outcome data

``` r
library(survival)
library(survminer)
#Scripts to read in the canine OS clinical metadata
#clinical metadata for all cases that received Rapamycin (mTOR inhibitor) in addition to standard of care therapy
Slides2Outcomes_Rapa_all <- read_csv("~/Dog/Slides2Outcomes_Rapa_all_new.csv")
```

    ## New names:
    ## Rows: 295 Columns: 24
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," chr
    ## (15): Study, Site, Name, breed, gender, Tumor Location, PH vs NPH, ALP, ... dbl
    ## (9): ...1, Patient ID, age, weight, PH, DFI, DFI_censor, Survival (days...
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
#clinical metadata for all cases that received standard of care therapy
Slides2Outcomes_SOC_all <- read_csv("~/Dog/Slides2Outcomes_SOC_all_new.csv")
```

    ## New names:
    ## Rows: 305 Columns: 24
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," chr
    ## (15): Study, Site, Name, breed, gender, Tumor Location, PH vs NPH, ALP, ... dbl
    ## (9): ...1, Patient ID, age, weight, PH, DFI, DFI_censor, Survival (days...
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
dat1 <- subset(Slides2Outcomes_Rapa_all,select = c("slide","Patient ID","Tumor Location","Site","age","weight","breed","gender","PH","ALP","Group","DFI","DFI_censor","Survival (days from sx)","Surv_censor"))
dat1$treatment = rep("Rapamycin", nrow(dat1))

dat2 <- subset(Slides2Outcomes_SOC_all,select = c("slide","Patient ID","Tumor Location","Site","age","weight","breed","gender","PH","ALP","Group","DFI","DFI_censor","Survival (days from sx)","Surv_censor"))
dat2$treatment = rep("SOC", nrow(dat2))

clindat_all <- rbind(dat1,dat2)
clindat_all <- as.data.frame(clindat_all)
clindat_all <- clindat_all[!duplicated(clindat_all$slide),]
rownames(clindat_all) <- substr(clindat_all$slide,1,19)
clindat_all <- clindat_all[,-1]
colnames(clindat_all)[c(11,12,13,14)] <- c("DFS_time","DFS_status","OS_time","OS_status")
clindat_all$ALP[clindat_all$ALP == "elevated"] <- "Elevated"
clindat_all$gender[clindat_all$`Patient ID` == "307"] <- "Spayed Female"
clindat_all$gender[grepl("Castrated",clindat_all$gender)] <- "Castrated Male"
clindat_all$gender[grepl("Spayed",clindat_all$gender)] <- "Spayed Female"
clindat_all$gender[grepl("Male Phenotype",clindat_all$gender)] <- "Intact Male"
clindat_all$gender[grepl("Female Phenotype",clindat_all$gender)] <- "Intact Female"
clindat_all <- clindat_all[!duplicated(clindat_all$`Patient ID`),]
cc1 <- clindat_all
rownames(cc1) <- as.character(clindat_all$`Patient ID`)
common <- intersect(names(imm_clus)[sample_type == "Primary"], rownames(cc1))
cc1 <- cc1[common,]
cc1$primary_immune_subtype <- as.character(imm_clus[sample_type == "Primary"][common])

#cc1$integrated = cc1$group
#cc1$integrated[cc1$group == "{cluster 1, 2}"] = cc1$immune_subtype[cc1$group == "{cluster 1, 2}"]
ccc1 <-cc1
#ccc1 <- ccc1[names(met_subtype_lung),]
#ccc1$met_subtype_lung <- met_subtype_lung
surv <- survfit(Surv(OS_time, OS_status) ~ primary_immune_subtype, data = ccc1)
diff <- survdiff(Surv(OS_time, OS_status) ~ primary_immune_subtype, data = ccc1)

p1 <- ggsurvplot(surv, data = ccc1,
                  legend.title = "",
                 legend = "right",
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                 palette = c("#88CCEE","#882255","#CC6677"),
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days from sx)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-13-1.png)

``` r
#ggsave(filename = "~/Dog/canine_OS_survfigure.pdf", plot=p1$plot, width = 9, height = 6)

surv <- survfit(Surv(DFS_time, DFS_status) ~ primary_immune_subtype, data = ccc1)
diff <- survdiff(Surv(DFS_time, DFS_status) ~ primary_immune_subtype, data = ccc1)

p1 <- ggsurvplot(surv, data = ccc1,
                  legend.title = "",
                 legend = "right",
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                 palette = c("#88CCEE","#882255","#CC6677"),
                  ggtheme = theme(text = element_text(size = 15)) + theme_bw() # Change ggplot2 theme
) + xlab("Time (days from sx)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-13-2.png)

``` r
#ggsave(filename = "~/Dog/canine_DFS_survfigure.pdf", plot=p1$plot, width = 9, height = 6)
```

# Multivaruate Cox proportional hazards analysis to account for confounding clinical factors

``` r
cph_dat <- data.frame(time = cc1$DFS_time,
subtype = cc1$primary_immune_subtype,
cytotoxicity_score = as.numeric(cyt_scores[common]),
status = cc1$DFS_status,
location = c("NPH","PH")[as.numeric(ccc1$PH)+1],
ALP = cc1$ALP,
age = scale(as.numeric(cc1$age)),
gender = cc1$gender,
weight = scale(cc1$weight),
treatment = c("SOC","Sirolimus+SOC")[as.numeric(cc1$treatment == "Rapamycin")+1])
cph_dat$gender[grepl("female", ignore.case = T, cph_dat$gender)] = "F"
cph_dat$gender[grepl("male", ignore.case = T, cph_dat$gender)] = "M"
cph_dat$ALP[cph_dat$ALP != "Elevated"] = "Normal"
#cph_dat$isFemale = as.numeric(cph_dat$isFemale == "F")
cph_dat$ALP <- relevel(as.factor(cph_dat$ALP), ref = "Normal")
cph_dat$treatment <- relevel(as.factor(cph_dat$treatment), ref = "SOC")
cph_dat$location <- relevel(as.factor(cph_dat$location), ref = "NPH")
cph_dat$gender <- relevel(as.factor(cph_dat$gender), ref = "M")
fit <- coxph(Surv(time, status) ~ location + ALP + age + gender + weight + treatment +  subtype, data = cph_dat)
p1 <- ggforest(fit, data = cph_dat,fontsize = 0.7, noDigits = 2) + theme(text = element_text(size = 15))
#ggsave(plot = p1, filename = "~/Dog/ggforest_DFS.pdf", width = 6, height = 8)


cph_dat <- data.frame(time = cc1$OS_time,
subtype = cc1$primary_immune_subtype,
cytotoxicity_score = as.numeric(cyt_scores[common]),
status = cc1$OS_status,
location = c("NPH","PH")[as.numeric(ccc1$PH)+1],
ALP = cc1$ALP,
age = scale(as.numeric(cc1$age)),
gender = cc1$gender,
weight = scale(cc1$weight),
treatment = c("SOC","Sirolimus+SOC")[as.numeric(cc1$treatment == "Rapamycin")+1])
cph_dat$gender[grepl("female", ignore.case = T, cph_dat$gender)] = "F"
cph_dat$gender[grepl("male", ignore.case = T, cph_dat$gender)] = "M"
cph_dat$ALP[cph_dat$ALP != "Elevated"] = "Normal"
#cph_dat$isFemale = as.numeric(cph_dat$isFemale == "F")
cph_dat$ALP <- relevel(as.factor(cph_dat$ALP), ref = "Normal")
cph_dat$treatment <- relevel(as.factor(cph_dat$treatment), ref = "SOC")
cph_dat$location <- relevel(as.factor(cph_dat$location), ref = "NPH")
cph_dat$gender <- relevel(as.factor(cph_dat$gender), ref = "M")
fit <- coxph(Surv(time, status) ~ location + ALP + age + gender + weight + treatment +  subtype, data = cph_dat)
p2 <-ggforest(fit, data = cph_dat,fontsize = 0.7, noDigits = 2) + theme(text = element_text(size = 15))

#ggsave(plot = p2, filename = "~/Dog/ggforest_OS.pdf", width = 6, height = 8)

fig <- ggarrange(p2,p1, ncol = 2, nrow = 1, labels = c("",""))
print(fig)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-14-1.png)

``` r
#ggsave(plot = fig, filename = "~/Dog/ggforest_cyt_figures.pdf",width = 12, height = 8)
```

#Plot TME characteristics of metastatic samples in long term vs short
term survivors

``` r
library(reshape)
cc1 <- clindat_all
rownames(cc1) <- as.character(clindat_all$`Patient ID`)

met_dd1 <- data.frame(surv_group = c("shorter survival","longer survival")[as.numeric(cc1[matched_cases$case[matched_cases$sample_type == "Metastatic"], "OS_time"] > median(cc1$OS_time[cc1$OS_status==1]))+1], os_status = as.numeric(cc1[matched_cases$case[matched_cases$sample_type == "Metastatic"], "OS_status"]), os_time = as.numeric(cc1[matched_cases$case[matched_cases$sample_type == "Metastatic"], "OS_time"]), met_subtype = matched_cases$immune_subtype[matched_cases$sample_type == "Metastatic"], sites = matched_cases$site2[matched_cases$sample_type == "Metastatic"], matched_cases[matched_cases$sample_type == "Metastatic",1:10])
met_dd1 <- met_dd1[!is.na(met_dd1$surv_group),]

ha <- columnAnnotation(
  group = met_dd1$surv_group,
  preservation = matched_cases$prep[matched_cases$sample_type == "Metastatic"],
  platform = matched_cases$source[matched_cases$sample_type == "Metastatic"],
  subtype = matched_cases$immune_subtype[matched_cases$sample_type == "Metastatic"],
  col = list(
               "preservation" = c("Frozen" = "#db6d00", 
                                  "FFPE" = "#924900"),
               "platform" = c("Nanostring" = "#525252",
                              "RNASeq" = "#F2F0F7"),
               "subtype" = c("ID" = "#88CCEE",
                             "IE" = "#882255",
                             "IE-ECM" = "#CC6677"),
             group = c("shorter survival"="#AE017E",
                       "longer survival"="#FDAE61")
    )
)
mat <- as.matrix(t(met_dd1[,6:ncol(met_dd1)]))
colnames(mat) <- matched_cases$case[matched_cases$sample_type == "Metastatic"]
colnames(mat) <- gsub("X","",colnames(mat))
colnames(mat) <- gsub("(\\.1)|(\\.2)|(\\.3)|(\\.4)|(\\.5)|(\\.6)","",colnames(mat))
Heatmap(mat, column_split = met_dd1$sites, bottom_annotation = ha, cluster_rows = F, cluster_column_slices = F,column_gap = unit(1, "mm"))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-15-1.png)

``` r
#pdf("~/Dog/met_sites_survival_association.pdf", width = 16, height = 4)
#Heatmap(mat, column_split = met_dd1$sites, bottom_annotation = ha, cluster_rows = F, cluster_column_slices = F,column_gap = unit(1, "mm"))
#dev.off()

#Heatmap(as.matrix(met_dd1[,6:ncol(met_dd1)]), row_split = met_dd1$surv_group, cluster_columns = F, cluster_row_slices = F, row_gap = unit(5, "mm"), right_annotation = ha2)
met_dd2 <- melt(met_dd1[,c(1,6:ncol(met_dd1))])
```

    ## Using surv_group as id variables

``` r
p1 <- ggplot(data = met_dd2, aes(x = variable, y = value, fill = surv_group)) + geom_boxplot() + theme_bw() + theme(text = element_text(size = 15), axis.text.x = element_text(angle = 90)) + stat_compare_means(aes(group = surv_group), label = "p.signif", method = "t.test") + ylim(c(-4,4))
print(p1)
```

    ## Warning: Removed 2 rows containing non-finite outside the scale range
    ## (`stat_boxplot()`).

    ## Warning: Removed 2 rows containing non-finite outside the scale range
    ## (`stat_compare_means()`).

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-15-2.png)

``` r
#ggsave(filename = "~/Dog/met_survival_association.pdf",plot = p1, width = 12, height = 7)
```

# Perform Differential expression + gene set enrichment analysis to characterize enriched hallmark pathways in each TME cluster

``` r
library(edgeR)
```

    ## Loading required package: limma

``` r
library(matrixStats)
```

    ## 
    ## Attaching package: 'matrixStats'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     count

``` r
library(fgsea)
library(tidyverse)
```

    ## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ## ✔ forcats   1.0.0     ✔ stringr   1.5.1
    ## ✔ lubridate 1.9.3     ✔ tidyr     1.3.0
    ## ✔ purrr     1.0.2

    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ matrixStats::count() masks dplyr::count()
    ## ✖ tidyr::expand()      masks reshape::expand()
    ## ✖ plotly::filter()     masks dplyr::filter(), stats::filter()
    ## ✖ dplyr::lag()         masks stats::lag()
    ## ✖ readr::parse_date()  masks curl::parse_date()
    ## ✖ plotly::rename()     masks reshape::rename(), dplyr::rename()
    ## ✖ lubridate::stamp()   masks reshape::stamp()
    ## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(msigdbr)
library(clusterProfiler)
```

    ## 
    ## Registered S3 methods overwritten by 'treeio':
    ##   method              from    
    ##   MRCA.phylo          tidytree
    ##   MRCA.treedata       tidytree
    ##   Nnode.treedata      tidytree
    ##   Ntip.treedata       tidytree
    ##   ancestor.phylo      tidytree
    ##   ancestor.treedata   tidytree
    ##   child.phylo         tidytree
    ##   child.treedata      tidytree
    ##   full_join.phylo     tidytree
    ##   full_join.treedata  tidytree
    ##   groupClade.phylo    tidytree
    ##   groupClade.treedata tidytree
    ##   groupOTU.phylo      tidytree
    ##   groupOTU.treedata   tidytree
    ##   is.rooted.treedata  tidytree
    ##   nodeid.phylo        tidytree
    ##   nodeid.treedata     tidytree
    ##   nodelab.phylo       tidytree
    ##   nodelab.treedata    tidytree
    ##   offspring.phylo     tidytree
    ##   offspring.treedata  tidytree
    ##   parent.phylo        tidytree
    ##   parent.treedata     tidytree
    ##   root.treedata       tidytree
    ##   rootnode.phylo      tidytree
    ##   sibling.phylo       tidytree
    ## clusterProfiler v4.6.2  For help: https://yulab-smu.top/biomedical-knowledge-mining-book/
    ## 
    ## If you use clusterProfiler in published research, please cite:
    ## T Wu, E Hu, S Xu, M Chen, P Guo, Z Dai, T Feng, L Zhou, W Tang, L Zhan, X Fu, S Liu, X Bo, and G Yu. clusterProfiler 4.0: A universal enrichment tool for interpreting omics data. The Innovation. 2021, 2(3):100141
    ## 
    ## Attaching package: 'clusterProfiler'
    ## 
    ## The following object is masked from 'package:purrr':
    ## 
    ##     simplify
    ## 
    ## The following object is masked from 'package:reshape':
    ## 
    ##     rename
    ## 
    ## The following object is masked from 'package:stats':
    ## 
    ##     filter

``` r
# Read raw counts
DOG2_Raw_RNA_SEQ_DATA <- read_csv("~/Dog/DOG2_Raw_RNA_SEQ_DATA.csv")
```

    ## Rows: 37952 Columns: 187
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr   (1): gene_id
    ## dbl (186): 102, 105, 106, 201, 202, 203, 204, 206, 207, 209, 210, 211, 212, ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

## IE
``` r
genes <- DOG2_Raw_RNA_SEQ_DATA$gene_id
genes <- sapply(genes, function(x) strsplit(x, split = "_")[[1]][1])
DOG2_Raw_RNA_SEQ_DATA$gene_id <- NULL
groups = ii1[colnames(DOG2_Raw_RNA_SEQ_DATA)]
groups = c("rest","IE")[as.numeric(groups == "IE")+1]
counts.DGEList <- DGEList(counts = DOG2_Raw_RNA_SEQ_DATA, genes = genes, group = as.factor(groups))
counts.keep <- filterByExpr(y = counts.DGEList)
counts.DGEList <- counts.DGEList[counts.keep, , keep.lib.sizes = FALSE]
counts.DGEList <- calcNormFactors(counts.DGEList)
counts.DGEList <- estimateDisp(counts.DGEList,design = model.matrix(~groups))
rest_IE.DGEExact <- exactTest(counts.DGEList, pair = c("rest","IE"))
msigdbr_df <- msigdbr(species = "Homo sapiens", category = "H")
msigdbr_list = split(x = msigdbr_df$gene_symbol, f = msigdbr_df$gs_name)
ranks <- rest_IE.DGEExact$table$logFC
names(ranks) <- counts.DGEList$genes$genes
ranks <- sort(ranks, decreasing = T)
gsea_res <- GSEA(geneList = ranks, TERM2GENE = subset(msigdbr_df, select = c("gs_name","gene_symbol")),pvalueCutoff = 1)
```

    ## preparing geneSet collections...
    ## GSEA analysis...

    ## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (0.12% of the list).
    ## The order of those tied genes will be arbitrary, which may produce unexpected results.

    ## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize,
    ## gseaParam, : There are duplicate gene names, fgsea may produce unexpected
    ## results.

    ## Warning in fgseaMultilevel(pathways = pathways, stats = stats, minSize =
    ## minSize, : For some pathways, in reality P-values are less than 1e-10. You can
    ## set the `eps` argument to zero for better estimation.

    ## leading edge analysis...
    ## done...

``` r
fgseaResTidy_IE <- gsea_res %>%
as_tibble() %>%
arrange(desc(NES))
p1 <- ggplot(fgseaResTidy_IE, aes(reorder(ID, NES), NES)) +
geom_col(aes(fill=p.adjust < 0.05)) +
coord_flip() +
labs(x="Pathway", y="Normalized Enrichment Score",title="Hallmark pathways NES from GSEA") + 
  theme_minimal()
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-16-1.png)
## IE-ECM
``` r
groups = ii1[colnames(DOG2_Raw_RNA_SEQ_DATA)]
groups = c("rest","IE-ECM")[as.numeric(groups == "IE-ECM")+1]
counts.DGEList <- DGEList(counts = DOG2_Raw_RNA_SEQ_DATA, genes = genes, group = as.factor(groups))
counts.keep <- filterByExpr(y = counts.DGEList)
counts.DGEList <- counts.DGEList[counts.keep, , keep.lib.sizes = FALSE]
counts.DGEList <- calcNormFactors(counts.DGEList)
counts.DGEList <- estimateDisp(counts.DGEList,design = model.matrix(~groups))
rest_IEECM.DGEExact <- exactTest(counts.DGEList, pair = c("rest","IE-ECM"))
msigdbr_df <- msigdbr(species = "Homo sapiens", category = "H")
msigdbr_list = split(x = msigdbr_df$gene_symbol, f = msigdbr_df$gs_name)


ranks <- rest_IEECM.DGEExact$table$logFC
names(ranks) <- counts.DGEList$genes$genes
ranks <- sort(ranks, decreasing = T)

gsea_res <- GSEA(geneList = ranks, TERM2GENE = subset(msigdbr_df, select = c("gs_name","gene_symbol")), pvalueCutoff = 1, nPermSimple=100000)
```

    ## preparing geneSet collections...

    ## GSEA analysis...

    ## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (0.11% of the list).
    ## The order of those tied genes will be arbitrary, which may produce unexpected results.

    ## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize,
    ## gseaParam, : There are duplicate gene names, fgsea may produce unexpected
    ## results.

    ## Warning in fgseaMultilevel(pathways = pathways, stats = stats, minSize =
    ## minSize, : For some pathways, in reality P-values are less than 1e-10. You can
    ## set the `eps` argument to zero for better estimation.

    ## leading edge analysis...

    ## done...

``` r
fgseaResTidy_IEECM <- gsea_res %>%
  as_tibble() %>%
  arrange(desc(NES))


p1 <- ggplot(fgseaResTidy_IEECM, aes(reorder(ID, NES), NES)) +
  geom_col(aes(fill=p.adjust < 0.05)) +
  coord_flip() +
  labs(x="Pathway", y="Normalized Enrichment Score",
       title="Hallmark pathways NES from GSEA") + 
  theme_minimal()
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-17-1.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/IEECM_enrichmentplot.pdf")
```
## ID
``` r

groups = ii1[colnames(DOG2_Raw_RNA_SEQ_DATA)]
groups = c("rest","ID")[as.numeric(groups == "ID")+1]
counts.DGEList <- DGEList(counts = DOG2_Raw_RNA_SEQ_DATA, genes = genes, group = as.factor(groups))
counts.keep <- filterByExpr(y = counts.DGEList)
counts.DGEList <- counts.DGEList[counts.keep, , keep.lib.sizes = FALSE]
counts.DGEList <- calcNormFactors(counts.DGEList)
counts.DGEList <- estimateDisp(counts.DGEList,design = model.matrix(~groups))
rest_ID.DGEExact <- exactTest(counts.DGEList, pair = c("rest","ID"))


msigdbr_df <- msigdbr(species = "Homo sapiens", category = "H")
msigdbr_list = split(x = msigdbr_df$gene_symbol, f = msigdbr_df$gs_name)


ranks <- rest_ID.DGEExact$table$logFC
names(ranks) <- counts.DGEList$genes$genes
ranks <- sort(ranks, decreasing = T)

gsea_res <- GSEA(geneList = ranks, TERM2GENE = subset(msigdbr_df, select = c("gs_name","gene_symbol")),pvalueCutoff = 1, nPermSimple = 100000)
```

    ## preparing geneSet collections...

    ## GSEA analysis...

    ## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize, gseaParam, : There are ties in the preranked stats (0.11% of the list).
    ## The order of those tied genes will be arbitrary, which may produce unexpected results.

    ## Warning in preparePathwaysAndStats(pathways, stats, minSize, maxSize,
    ## gseaParam, : There are duplicate gene names, fgsea may produce unexpected
    ## results.

    ## Warning in fgseaMultilevel(pathways = pathways, stats = stats, minSize =
    ## minSize, : For some pathways, in reality P-values are less than 1e-10. You can
    ## set the `eps` argument to zero for better estimation.

    ## leading edge analysis...

    ## done...

``` r
fgseaResTidy_ID <- gsea_res %>%
  as_tibble() %>%
  arrange(desc(NES))



p1 <- ggplot(fgseaResTidy_ID, aes(reorder(ID, NES), NES)) +
  geom_col((aes(fill=p.adjust < 0.05))) +
  coord_flip() +
  labs(x="Pathway", y="Normalized Enrichment Score",
       title="Hallmark pathways NES from GSEA") + 
  theme_minimal()
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-18-1.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/ID_enrichmentplot.pdf")
```

# Generate combined dotplot depicting differential expression + gsea
results

``` r
NES_IE <- fgseaResTidy_IE$NES
names(NES_IE) <- fgseaResTidy_IE$ID
NES_IEECM <- fgseaResTidy_IEECM$NES
names(NES_IEECM) <- fgseaResTidy_IEECM$ID
NES_ID <- fgseaResTidy_ID$NES
names(NES_ID) <- fgseaResTidy_ID$NES
NES_ID <- fgseaResTidy_ID$NES
names(NES_ID) <- fgseaResTidy_ID$ID
NES_mat <- cbind(NES_IE, cbind(NES_IEECM[names(NES_IE)], NES_ID[names(NES_IE)]))
rownames(NES_mat) <- names(NES_IE)
lpval_IE <- -log10(fgseaResTidy_IE$p.adjust)
names(lpval_IE) <- fgseaResTidy_IE$ID
lpval_IEECM <- -log10(fgseaResTidy_IEECM$p.adjust)
names(lpval_IEECM) <- fgseaResTidy_IEECM$ID
lpval_ID <- -log10(fgseaResTidy_ID$p.adjust)
names(lpval_ID) <- fgseaResTidy_ID$ID
pval_mat <- cbind(lpval_IE, cbind(lpval_IEECM[names(lpval_IE)],lpval_ID[names(lpval_IE)]))
rownames(pval_mat) <- names(lpval_IE)
colnames(NES_mat) <- c("IE","IE-ECM","ID")

NES_mat <- NES_mat[names(NES_IEECM),]
pval_mat <- pval_mat[names(NES_IEECM),]

col_fun = circlize::colorRamp2(c(-1, 0, 1), c("navyblue", "darkmagenta", "yellow"))

layer_fun = function(j, i, x, y, w, h, fill){
          grid.rect(x = x, y = y, width = w, height = h, 
                    gp = gpar(col = NA, fill = NA))
          grid.circle(x=x,y=y,r= pindex(pval_mat, i, j)/10 * unit(2, "mm"),
                      gp = gpar(fill = col_fun(pindex(NES_mat, i, j)), col = NA))}


lgd_list = list(
    Legend( labels = c(0,2.5,5,7.5,10), title = "-log10(adjusted P-value)",
            graphics = list(
              function(x, y, w, h) grid.circle(x = x, y = y, r = 0 * unit(2, "mm"),
                                               gp = gpar(fill = "black")),
              function(x, y, w, h) grid.circle(x = x, y = y, r = 0.25 * unit(2, "mm"),
                                               gp = gpar(fill = "black")),
              function(x, y, w, h) grid.circle(x = x, y = y, r = 0.5 * unit(2, "mm"),
                                               gp = gpar(fill = "black")),
              function(x, y, w, h) grid.circle(x = x, y = y, r = 0.75 * unit(2, "mm"),
                                               gp = gpar(fill = "black")),
              function(x, y, w, h) grid.circle(x = x, y = y, r = 1 * unit(2, "mm"),
                                               gp = gpar(fill = "black")))
            ))

set.seed(123)    
hp<- Heatmap(NES_mat,
        heatmap_legend_param=list(title="NES"),
        col=col_fun,
        rect_gp = gpar(type = "none"),
        layer_fun = layer_fun,
        row_names_gp = gpar(fontsize = 7.5),
        cluster_rows = F,
        cluster_columns = F)

draw( hp, annotation_legend_list = lgd_list)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-19-1.png)

# Read human data

# TARGET

``` r
library("org.Hs.eg.db") 
```

    ## Loading required package: AnnotationDbi

    ## Loading required package: stats4

    ## Loading required package: BiocGenerics

    ## 
    ## Attaching package: 'BiocGenerics'

    ## The following objects are masked from 'package:lubridate':
    ## 
    ##     intersect, setdiff, union

    ## The following object is masked from 'package:limma':
    ## 
    ##     plotMA

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     combine, intersect, setdiff, union

    ## The following objects are masked from 'package:stats':
    ## 
    ##     IQR, mad, sd, var, xtabs

    ## The following objects are masked from 'package:base':
    ## 
    ##     anyDuplicated, aperm, append, as.data.frame, basename, cbind,
    ##     colnames, dirname, do.call, duplicated, eval, evalq, Filter, Find,
    ##     get, grep, grepl, intersect, is.unsorted, lapply, Map, mapply,
    ##     match, mget, order, paste, pmax, pmax.int, pmin, pmin.int,
    ##     Position, rank, rbind, Reduce, rownames, sapply, setdiff, sort,
    ##     table, tapply, union, unique, unsplit, which.max, which.min

    ## Loading required package: Biobase

    ## Welcome to Bioconductor
    ## 
    ##     Vignettes contain introductory material; view with
    ##     'browseVignettes()'. To cite Bioconductor, see
    ##     'citation("Biobase")', and for packages 'citation("pkgname")'.

    ## 
    ## Attaching package: 'Biobase'

    ## The following objects are masked from 'package:matrixStats':
    ## 
    ##     anyMissing, rowMedians

    ## Loading required package: IRanges

    ## Loading required package: S4Vectors

    ## 
    ## Attaching package: 'S4Vectors'

    ## The following object is masked from 'package:clusterProfiler':
    ## 
    ##     rename

    ## The following objects are masked from 'package:lubridate':
    ## 
    ##     second, second<-

    ## The following object is masked from 'package:tidyr':
    ## 
    ##     expand

    ## The following object is masked from 'package:plotly':
    ## 
    ##     rename

    ## The following objects are masked from 'package:reshape':
    ## 
    ##     expand, rename

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     first, rename

    ## The following objects are masked from 'package:base':
    ## 
    ##     expand.grid, I, unname

    ## 
    ## Attaching package: 'IRanges'

    ## The following object is masked from 'package:clusterProfiler':
    ## 
    ##     slice

    ## The following object is masked from 'package:lubridate':
    ## 
    ##     %within%

    ## The following object is masked from 'package:purrr':
    ## 
    ##     reduce

    ## The following object is masked from 'package:plotly':
    ## 
    ##     slice

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     collapse, desc, slice

    ## 
    ## Attaching package: 'AnnotationDbi'

    ## The following object is masked from 'package:clusterProfiler':
    ## 
    ##     select

    ## The following object is masked from 'package:plotly':
    ## 
    ##     select

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     select

    ## 

``` r
TARGET_OS_star_counts <- read_delim("~/Downloads/TARGET-OS.star_counts.tsv", 
delim = "\t", escape_double = FALSE, 
trim_ws = TRUE)
```

    ## Rows: 60487 Columns: 89

    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: "\t"
    ## chr  (1): Ensembl_ID
    ## dbl (88): TARGET-40-PASUUH-01A, TARGET-40-PAUTWB-01A, TARGET-40-PAKUZU-01A, ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
genes <- TARGET_OS_star_counts$Ensembl_ID
genes <- sapply(genes, function(x) strsplit(x, split = "\\.")[[1]][1])
symbols <- mapIds(org.Hs.eg.db, keys = genes, keytype = "ENSEMBL", column="SYMBOL")
```

    ## 'select()' returned 1:many mapping between keys and columns

``` r
#mart <- useDataset("hsapiens_gene_ensembl", useMart("ensembl"))
#G_list <- getBM(filters= "ensembl_gene_id", attributes= c("ensembl_gene_id","hgnc_symbol"),values=genes,mart= mart)
TARGET_OS_star_counts <- as.data.frame(TARGET_OS_star_counts)
TARGET_OS_star_counts <- TARGET_OS_star_counts[!is.na(symbols),]
expr_human <- aggregate.data.frame(TARGET_OS_star_counts[,2:ncol(TARGET_OS_star_counts)], by = list(symbols[!is.na(symbols)]), FUN = mean, na.rm=T)
rownames(expr_human) <- expr_human$Group.1
expr_human <- expr_human[,-1]

TARGET_OS_clinical <- read_delim("~/Downloads/TARGET-OS.clinical.tsv", delim = "\t", escape_double = FALSE, trim_ws = TRUE)
```

    ## Rows: 527 Columns: 31
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: "\t"
    ## chr (23): sample_id, TARGET USI, Gender, Race, Ethnicity, First Event, Time ...
    ## dbl  (8): Age at Diagnosis in Days, Overall Survival Time in Days, Year of D...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
TARGET_OS_clinical <- as.data.frame(TARGET_OS_clinical)
TARGET_OS_clinical <- TARGET_OS_clinical[!duplicated(TARGET_OS_clinical$sample_id),]
rownames(TARGET_OS_clinical) <- TARGET_OS_clinical$sample_id

common <- intersect(rownames(TARGET_OS_clinical), colnames(expr_human))
ee <- expr_human[,common]
human_clin <- TARGET_OS_clinical[common,]


set.seed(2425)
#ccommon = intersect(rownames(ee), intersect(rownames(expr_human2),rownames(expr_human3)))
cellprop_human <- t(MCPcounter.estimate(expression = ee, featuresType = "HUGO_symbols", probesets = read.table("~/Dog/MCP_counter_markers.csv")))
cellprop_human <- apply(cellprop_human, 2, scale)
#rownames(cellprop_primary) <- colnames(Canine_RNA_SEQ_TPM_All_data)
km.res <- kmeans(cellprop_human, centers = 3, nstart = 100, iter.max = 500)
imm_clus_target <- rep("ID", nrow(cellprop_human))
imm_clus_target[km.res$cluster == 3] <- "IE"
imm_clus_target[km.res$cluster == 1] <- "IE-ECM"
#imm_clus <- factor(imm_clus, levels = c("IE","IE fibrotic", "ID"))
Heatmap(cellprop_human, row_split = imm_clus_target, cluster_columns = F, cluster_row_slices = FALSE, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-20-1.png)

``` r
names(imm_clus_target) <- colnames(ee)
human_clin <- human_clin[names(imm_clus_target),]
human_clin$immune_subtype <- imm_clus_target
hc <- data.frame(OS_time = as.numeric(human_clin$`Overall Survival Time in Days`), OS_status = as.numeric(human_clin$`Vital Status` == "Dead"),
                 DFS_time = as.numeric(human_clin$`Time to First Event in Days`), DFS_status = as.numeric(human_clin$`First Event` == "Relapse"),
                 immune_subtype = human_clin$immune_subtype,
                 met_at_daig = toupper(human_clin$`Disease at diagnosis`),
                 age = human_clin$`Age at Diagnosis in Days`/365,
                 gender = c("M","F")[as.numeric(human_clin$Gender == "Female")+1])
hc$necrosis <- NA
hc$necrosis[!is.na(human_clin$`Percent necrosis`)] <- c("0-90","91-100")[as.numeric(as.numeric(human_clin$`Percent necrosis`[!is.na(human_clin$`Percent necrosis`)]) > 90)+1]
```

    ## Warning: NAs introduced by coercion

``` r
hc$necrosis[!is.na(human_clin$`Histologic response`)] <-human_clin$`Histologic response`[!is.na(human_clin$`Histologic response`)]
hc$necrosis[hc$necrosis %in% c("91-100","Stage 3/4","98","95") & !is.na(hc$necrosis)] <- ">90%"
hc$necrosis[!(hc$necrosis %in% c(">90%")) & !is.na(hc$necrosis)] <- "<=90%"

hc$met_at_daig[hc$met_at_daig == "METASTATIC (CONFIRMED)"] = "METASTATIC"
hc$met_at_daig[hc$met_at_daig == "NON-METASTATIC (CONFIRMED)"] = "NON-METASTATIC"

surv <- survfit(Surv(OS_time, OS_status) ~ immune_subtype, data = hc)
diff <- survdiff(Surv(OS_time, OS_status) ~ immune_subtype, data = hc)

p1 <- ggsurvplot(surv, data = hc,
                  
                  legend.title = "",
                 legend = "right",
                 #legend.labs = c("ID","IE","IE fibrotic"),
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-20-2.png)

``` r
 surv <- survfit(Surv(DFS_time, DFS_status) ~ immune_subtype, data = hc)
 diff <- survdiff(Surv(DFS_time, DFS_status) ~ immune_subtype, data = hc)
# 
 p1 <- ggsurvplot(surv, data = hc,
                   
                   legend.title = "",
                  legend = "right",
                  #legend.labs = c("ID","IE","IE fibrotic"),
                   conf.int = F,
                   pval = TRUE,
                   risk.table = TRUE,
                   tables.height = 0.2,
                   tables.theme = theme_cleantable(),
                   risk.table.y.text = FALSE,
                  pval.coord = c(0, 0.03),
                   # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                   # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                   ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
 ) + xlab("Time (days)")
 print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-20-3.png)

# GSE21257

``` r
expr_human2 <- read_csv("~/Dog/GSE21257_expr.csv")
```

    ## New names:
    ## Rows: 30549 Columns: 54
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," chr
    ## (1): ...1 dbl (53): GSM530667, GSM530899, GSM531283, GSM531284, GSM531285,
    ## GSM531286, ...
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
expr_human2 <- as.data.frame(expr_human2)
expr_human2 <- aggregate.data.frame(expr_human2[,2:ncol(expr_human2)], by = list(as.character(expr_human2[,1])), FUN = mean, na.rm = T)
rownames(expr_human2) <- expr_human2[,1]
expr_human2 <- expr_human2[,-1]
expr_human2 <- as.matrix(expr_human2)

set.seed(2425)
cellprop_human2 <- t(MCPcounter.estimate(expression = expr_human2, featuresType = "HUGO_symbols", probesets = read.table("~/Dog/MCP_counter_markers.csv")))
cellprop_human2 <- apply(cellprop_human2, 2, scale)
km.res <- kmeans(cellprop_human2, centers = 3, nstart = 100, iter.max = 500)
imm_clus2 <- rep("ID", nrow(cellprop_human2))
imm_clus2[km.res$cluster == 1] <- "IE"
imm_clus2[km.res$cluster == 3] <- "IE-ECM"
Heatmap(cellprop_human2, row_split = imm_clus2, cluster_columns = F, cluster_row_slices = F, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-21-1.png)

``` r
names(imm_clus2) <- colnames(expr_human2)
human_clin2 <- read_delim("~/Dog/GSE21257_pheno.tsv", delim = "\t", escape_double = FALSE, trim_ws = TRUE)
```

    ## New names:
    ## Rows: 53 Columns: 49
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "\t" chr
    ## (40): ...1, title, geo_accession, status, submission_date, last_update_d... dbl
    ## (8): channel_count, taxid_ch1, contact_zip/postal_code, data_row_count,... lgl
    ## (1): ...45
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`
    ## • `` -> `...45`

``` r
human_clin2 <- as.data.frame(human_clin2)
rownames(human_clin2) <- human_clin2$...1
human_clin2 <- human_clin2[,-1]
human_clin2 <- human_clin2[names(imm_clus2),]
human_clin2$`age:ch1` <- sapply(human_clin2$`age:ch1`, function(x) as.numeric(strsplit(x, split = " ")[[1]][1])/12)
human_clin2$immune_subtype <- as.character(imm_clus2)
hc2 <- data.frame(OS_time = as.numeric(human_clin2$OS_time*30), OS_status = as.numeric(human_clin2$OS_status),
                 DFS_time = as.numeric(human_clin2$MFS_time*30), DFS_status = as.numeric(human_clin2$met_status),
                 immune_subtype = human_clin2$immune_subtype,
                 met_at_daig = human_clin2$`group:ch1`,
                 age = human_clin2$`age:ch1`,
                 gender = human_clin2$`gender:ch1`)
hc2$met_at_daig = c("NON-METASTATIC","METASTATIC")[as.numeric(hc2$met_at_daig == "Metastases present at diagnosis")+1]
hc2 <- hc2[!is.na(hc2$OS_status) & !is.na(hc2$DFS_status),]
hc2$necrosis <- NA
surv <- survfit(Surv(OS_time, OS_status) ~ immune_subtype, data = hc2)
diff <- survfit(Surv(OS_time, OS_status) ~ immune_subtype, data = hc2)

p1 <- ggsurvplot(surv, data = hc2,
                  
                  legend.title = "",
                 legend = "right",
                 legend.labs = c("ID","IE","IE-ECM"),
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-21-2.png)

``` r
surv <- survfit(Surv(DFS_time, DFS_status) ~ immune_subtype, data = hc2)
diff <- survfit(Surv(DFS_time, DFS_status) ~ immune_subtype, data = hc2)

p1 <- ggsurvplot(surv, data = hc2,
                  legend.title = "",
                 legend = "right",
                 #legend.labs = c("ID","IE","IE fibrotic"),
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-21-3.png)

# GSE39055

``` r
expr_human3 <- read_csv("~/Dog/GSE39055_expr.csv")
```

    ## New names:
    ## Rows: 29377 Columns: 38
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," chr
    ## (1): ...1 dbl (37): GSM954790, GSM954791, GSM954792, GSM954793, GSM954794,
    ## GSM954795, ...
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
expr_human3 <- as.data.frame(expr_human3)
expr_human3 <- aggregate.data.frame(expr_human3[,2:ncol(expr_human3)], by = list(as.character(expr_human3[,1])), FUN = mean, na.rm = T)
rownames(expr_human3) <- expr_human3[,1]
expr_human3 <- expr_human3[,-1]
#expr_human3 <- as.matrix(log2(expr_human3+1))

set.seed(2425)
cellprop_human3 <- t(MCPcounter.estimate(expression = expr_human3, featuresType = "HUGO_symbols",probesets = read.table("~/Dog/MCP_counter_markers.csv")))
cellprop_human3 <- apply(cellprop_human3, 2, scale)
km.res <- kmeans(cellprop_human3, centers = 3, nstart = 100, iter.max = 500)
imm_clus3 <- rep("ID", nrow(cellprop_human3))
imm_clus3[km.res$cluster == 1] <- "IE"
imm_clus3[km.res$cluster == 2] <- "IE-ECM"
#imm_clus3 <- factor(imm_clus3, levels = c("IE", "IE fibrotic", "ID"))
Heatmap(cellprop_human3, row_split = imm_clus3, cluster_columns = F, cluster_row_slices = F, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-22-1.png)

``` r
names(imm_clus3) <- colnames(expr_human3)

human_clin3 <- read_delim("~/Dog/GSE39055_pheno.tsv", delim = "\t", escape_double = FALSE, trim_ws = TRUE)
```

    ## Rows: 37 Columns: 51
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: "\t"
    ## chr (45): title, Metastatic_at_Diag, geo_accession, status, submission_date,...
    ## dbl  (6): channel_count, taxid_ch1, contact_zip/postal_code, data_row_count,...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
human_clin3 <- as.data.frame(human_clin3)
rownames(human_clin3) <- human_clin3$geo_accession

human_clin3 <- human_clin3[names(imm_clus3),]
human_clin3$immune_subtype <- as.character(imm_clus3)
human_clin3$`age:ch1` <- sapply(human_clin3$`age:ch1`, function(x) as.numeric(strsplit(x, split = " ")[[1]][1]))
#human_clin3[human_clin3$`time until first recurrence or latest follow-up (months):ch1` > 0,]
hc3 <- data.frame(OS_time = NA, OS_status = NA,
                 DFS_time = as.numeric(human_clin3$`time until first recurrence or latest follow-up (months):ch1`*30), DFS_status = as.numeric(human_clin3$`recurrence:ch1` == "Y"),
                 immune_subtype = human_clin3$immune_subtype, met_at_daig = human_clin3$Metastatic_at_Diag,
                 age = human_clin3$`age:ch1`,
                 gender = c("M","F")[as.numeric(human_clin3$`gender:ch1` == "female")+1])

hc3$necrosis <- c("<=90%",">90%")[as.numeric(human_clin3$necrosis =="Y")+1]

surv <- survfit(Surv(DFS_time, DFS_status) ~ immune_subtype, data = hc3)
diff <- survdiff(Surv(DFS_time, DFS_status) ~ immune_subtype, data = hc3)

p1 <- ggsurvplot(surv, data = hc3,
                  legend.title = "",
                 legend = "right",
                 #legend.labs = c("ID","IE","IE fibrotic"),
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-22-2.png)

# GSE16091

``` r
expr_human5 <- read_csv("~/Dog/GSE16091_expr.csv")
```

    ## Rows: 22283 Columns: 35
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr  (1): gene
    ## dbl (34): GSM402718, GSM402719, GSM402720, GSM402721, GSM402722, GSM402723, ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
expr_human5 <- as.data.frame(expr_human5)
expr_human5 <- aggregate.data.frame(expr_human5[,2:ncol(expr_human5)], by = list(as.character(expr_human5[,1])), FUN = mean, na.rm = T)
rownames(expr_human5) <- expr_human5[,1]
expr_human5 <- expr_human5[,-1]

set.seed(2425)
cellprop_human5 <- t(MCPcounter.estimate(expression = expr_human5, featuresType = "HUGO_symbols", probesets = read.table("~/Dog/MCP_counter_markers.csv")))
cellprop_human5 <- apply(cellprop_human5, 2, scale)
#rownames(cellprop_primary) <- colnames(Canine_RNA_SEQ_TPM_All_data)
km.res <- kmeans(cellprop_human5, centers = 3, nstart = 100, iter.max = 500)
imm_clus5 <- rep("ID", nrow(cellprop_human5))
imm_clus5[km.res$cluster == 3] <- "IE"
imm_clus5[km.res$cluster == 2] <- "IE-ECM"
#imm_clus3 <- factor(imm_clus3, levels = c("IE", "IE fibrotic", "ID"))
Heatmap(cellprop_human5, row_split = imm_clus5, cluster_columns = F, cluster_row_slices = F, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-23-1.png)

``` r
names(imm_clus5) <- colnames(expr_human5)

human_clin5 <- read_delim("~/Dog/GSE16091_pheno.tsv", delim = "\t", escape_double = FALSE, trim_ws = TRUE)
```

    ## Rows: 34 Columns: 35
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: "\t"
    ## chr (30): title, geo_accession, status, submission_date, last_update_date, t...
    ## dbl  (5): channel_count, taxid_ch1, contact_zip/postal_code, data_row_count,...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
human_clin5 <- as.data.frame(human_clin5)
rownames(human_clin5) <- human_clin5$geo_accession

human_clin5 <- human_clin5[names(imm_clus5),]
human_clin5$immune_subtype <- as.character(imm_clus5)
#human_clin3[human_clin3$`time until first recurrence or latest follow-up (months):ch1` > 0,]
hc5 <- data.frame(OS_time = as.numeric(human_clin5$`days-followup:ch1`), OS_status = as.numeric(human_clin5$`alive-or-dead:ch1` == "D"),
                 DFS_time = NA, DFS_status = NA,
                 immune_subtype = human_clin5$immune_subtype, met_at_daig = NA,
                 age = NA,
                 gender = NA,
                 necrosis = NA)

surv <- survfit(Surv(OS_time, OS_status) ~ immune_subtype, data = hc5)
diff <- survdiff(Surv(OS_time, OS_status) ~ immune_subtype, data = hc5)

p1 <- ggsurvplot(surv, data = hc5,
                  legend.title = "",
                 legend = "right",
                 #legend.labs = c("ID","IE","IE fibrotic"),
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-23-2.png)

# GSE30699

``` r
expr_human6 <- read_csv("~/Dog/GSE30699_expr.csv")
```

    ## Rows: 48701 Columns: 108
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr   (1): gene
    ## dbl (107): GSM761349, GSM761350, GSM761351, GSM761352, GSM761353, GSM761354,...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
human_clin6 <- read_delim("~/Dog/GSE30699_pheno.tsv", delim = "\t", escape_double = FALSE, trim_ws = TRUE)
```

    ## Rows: 107 Columns: 37
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: "\t"
    ## chr (34): title, geo_accession, status, submission_date, last_update_date, t...
    ## dbl  (3): channel_count, taxid_ch1, data_row_count
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
expr_human6 <- as.data.frame(expr_human6)
expr_human6 <- aggregate.data.frame(expr_human6[,2:ncol(expr_human6)], by = list(as.character(expr_human6[,1])), FUN = mean, na.rm = T)
rownames(expr_human6) <- expr_human6[,1]
expr_human6 <- expr_human6[,-1]
expr_human6 <- expr_human6[, grepl("biopsy",human_clin6$source_name_ch1)]

set.seed(2425)
cellprop_human6 <- t(MCPcounter.estimate(expression = expr_human6, featuresType = "HUGO_symbols", probesets = read.table("~/Dog/MCP_counter_markers.csv")))
cellprop_human6 <- apply(cellprop_human6, 2, scale)
km.res <- kmeans(cellprop_human6, centers = 3, nstart = 100, iter.max = 500)
imm_clus6 <- rep("ID", nrow(cellprop_human6))
imm_clus6[km.res$cluster == 3] <- "IE"
imm_clus6[km.res$cluster == 1] <- "IE-ECM"
Heatmap(cellprop_human6, row_split = imm_clus6, cluster_columns = F, cluster_row_slices = F, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-24-1.png)

``` r
names(imm_clus6) <- colnames(expr_human6)

hc6 <- data.frame(OS_time = NA, OS_status = NA,
                 DFS_time = NA, DFS_status = NA,
                 immune_subtype = imm_clus6, met_at_daig = NA,
                 age = NA,
                 gender =  NA,
                 necrosis = NA)
```

# GSE32981

``` r
expr_human7 <- read_csv("~/Dog/GSE32981_expr.csv")
```

    ## Rows: 12513 Columns: 24
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr  (1): Group.1
    ## dbl (23): GSM816942, GSM816943, GSM816944, GSM816945, GSM816946, GSM816947, ...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
expr_human7 <- expr_human7[!is.na(expr_human7$Group.1),]
expr_human7 <- as.data.frame(expr_human7)
rownames(expr_human7) <- expr_human7$Group.1
expr_human7 <- expr_human7[,-1]
GSE32981_pheno <- read_delim("~/Dog/GSE32981_pheno.tsv", delim = "\t", escape_double = FALSE, trim_ws = TRUE)
```

    ## Rows: 23 Columns: 42
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: "\t"
    ## chr (36): title, geo_accession, status, submission_date, last_update_date, t...
    ## dbl  (6): channel_count, taxid_ch1, contact_zip/postal_code, data_row_count,...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
set.seed(2425)
cellprop_human7 <- t(MCPcounter.estimate(expression = expr_human7, featuresType = "HUGO_symbols", probesets = read.table("~/Dog/MCP_counter_markers.csv")))
cellprop_human7 <- apply(cellprop_human7, 2, scale)
km.res <- kmeans(cellprop_human7, centers = 3, nstart = 100, iter.max = 500)
imm_clus7 <- rep("ID", nrow(cellprop_human7))
imm_clus7[km.res$cluster == 1] <- "IE"
imm_clus7[km.res$cluster == 3] <- "IE-ECM"
Heatmap(cellprop_human7, row_split = imm_clus7, cluster_columns = F, cluster_row_slices = F, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-25-1.png)

``` r
names(imm_clus7) <- colnames(expr_human7)
sample_type_7 <- c("Primary","Met")[as.numeric(GSE32981_pheno$`origin:ch1` == "Met")+1]
treatment_stat7 <- c("naive","treated")[as.numeric(GSE32981_pheno$Treated == "Yes")+1]
hc7 <- data.frame(OS_time = NA, OS_status = NA,
                 DFS_time = as.numeric(GSE32981_pheno$MFS)*30, DFS_status = as.numeric(GSE32981_pheno$`developed metastases:ch1` == "Yes"),
                 immune_subtype = imm_clus7, met_at_daig = as.numeric(GSE32981_pheno$MFS) == 0 & !is.na(as.numeric(GSE32981_pheno$MFS)),
                 age = as.numeric(GSE32981_pheno$`age:ch1`),
                 gender =  c("M","F")[as.numeric(GSE32981_pheno$`gender:ch1` == "Female")+1],
                 necrosis = GSE32981_pheno$necrosis)
hc7$met_at_daig <- c("NON-METASTATIC","METASTATIC")[as.numeric(hc7$met_at_daig & !is.na(hc7$met_at_daig)) +1]
```

# GSE33383

``` r
expr_human4 <- read_csv("~/Dog/GSE33383_expr.csv")
```

    ## New names:
    ## Rows: 30549 Columns: 100
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," chr
    ## (1): ...1 dbl (99): GSM717846, GSM717847, GSM717848, GSM717849, GSM717850,
    ## GSM717851, ...
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
expr_human4 <- as.data.frame(expr_human4)
expr_human4 <- aggregate.data.frame(expr_human4[,2:ncol(expr_human4)], by = list(as.character(expr_human4[,1])), FUN = mean, na.rm = T)
rownames(expr_human4) <- expr_human4[,1]
expr_human4 <- expr_human4[,-1]
GSE33383_pheno <- read_delim("~/Dog/GSE33383_pheno.tsv", delim = "\t", escape_double = FALSE, trim_ws = TRUE)
```

    ## Rows: 99 Columns: 50
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: "\t"
    ## chr (46): title, geo_accession, status, submission_date, last_update_date, t...
    ## dbl  (3): channel_count, taxid_ch1, data_row_count
    ## lgl  (1): description.1
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
expr_human4 <- expr_human4[,grepl("pre-chemotherapy biopsy", GSE33383_pheno$source_name_ch1)]
GSE33383_pheno <-GSE33383_pheno[grepl("pre-chemotherapy biopsy", GSE33383_pheno$source_name_ch1),]

set.seed(2425)
cellprop_human4 <- t(MCPcounter.estimate(expression = expr_human4, featuresType = "HUGO_symbols"))
cellprop_human4 <- apply(cellprop_human4, 2, scale)
#rownames(cellprop_primary) <- colnames(Canine_RNA_SEQ_TPM_All_data)
km.res <- kmeans(cellprop_human4, centers = 3, nstart = 100, iter.max = 500)
imm_clus4 <- rep("ID", nrow(cellprop_human4))
imm_clus4[km.res$cluster == 1] <- "IE"
imm_clus4[km.res$cluster == 3] <- "IE-ECM"
imm_clus4 <- factor(imm_clus4, levels = c("IE", "IE-ECM", "ID"))
ha = rowAnnotation(
    met_within_5yrs = GSE33383_pheno$`metastasis within 5yrs:ch1`,
    col = list("met_within_5yrs" = c("no" = "green", "yes" =  "yellow")
    )
)

names(imm_clus4) <- colnames(expr_human4)

hc44 <- data.frame(OS_time = NA, OS_status = NA,
                 DFS_time = NA, DFS_status = NA,
                 immune_subtype = imm_clus4, met_at_daig = rep("NON-METASTATIC", length(imm_clus4)),
                 age = sapply(GSE33383_pheno$`age:ch1`, function(x) as.numeric(strsplit(x," ")[[1]][1])/12),
                 gender =  c("M","F")[as.numeric(GSE33383_pheno$`gender:ch1` == "F")+1],
                 necrosis = NA)
```

    ## Warning in FUN(X[[i]], ...): NAs introduced by coercion

``` r
ct <- colnames(cellprop_human4)
```

# GSE152048

``` r
bulk_exp_scOS <- read_csv("~/Dog/GSE152048_merged_expr.csv")
```

    ## New names:
    ## Rows: 32864 Columns: 12
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," chr
    ## (1): ...1 dbl (11): BC2, BC3, BC5, BC6, BC10, BC11, BC16, BC17, BC20, BC21,
    ## BC22
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
bulk_exp_scOS <- as.data.frame(bulk_exp_scOS)
rownames(bulk_exp_scOS) <- bulk_exp_scOS$...1
bulk_exp_scOS <- bulk_exp_scOS[,-1]
cellprop_human_scOS <- t(MCPcounter.estimate(expression = bulk_exp_scOS, featuresType = "HUGO_symbols"))
cellprop_human_scOS <- apply(cellprop_human_scOS, 2, scale)
newcolnames <- sapply(colnames(cellprop_human_scOS), function(x) gsub(" ","_",x))
names(newcolnames) <- NULL
colnames(cellprop_human_scOS) <- newcolnames
sc_sample_type <- c("Primary","Primary","Primary","Primary","Met","Recurrent","Primary","Primary","Met","Recurrent","Primary")
```

# Build a TME subtype classifier to classify TME of new (human) tumor
samples

``` r
library(caret)
```

    ## Loading required package: lattice

    ## 
    ## Attaching package: 'lattice'

    ## The following object is masked from 'package:clusterProfiler':
    ## 
    ##     dotplot

    ## 
    ## Attaching package: 'caret'

    ## The following object is masked from 'package:purrr':
    ## 
    ##     lift

    ## The following object is masked from 'package:survival':
    ## 
    ##     cluster

``` r
library(e1071)
library(naivebayes)
```

    ## naivebayes 0.9.7 loaded

``` r
set.seed(268548)

df <- cbind(data.frame(rbind(cellprop_rnaseq[sample_type == "Primary",], cellprop_nano[sstatus == "Primary Tumor",])), imm_clus = c(imm_clus[sample_type=="Primary"],imm_clus_nano[sstatus=="Primary Tumor"]))
df$imm_clus <- as.factor(df$imm_clus)
colnames(df) <- gsub("\\.","_",colnames(df))
df <- df[,colnames(df)[c(1,3:ncol(df))]]
ind <- c(sample(which(df$imm_clus == "IE"),size = 50,replace = T), sample(which(df$imm_clus == "IE-ECM"),size = 50,replace = T), sample(which(df$imm_clus == "ID"),size = 50,replace = T))

#apply data augmentation
cp1 <- t(apply(df[ind,1:(ncol(df))-1],1,function(x) x + rnorm((ncol(df))-1, mean = 0, sd = 0.01)))
df1 <- cbind(data.frame(cp1), df$imm_clus[ind])
colnames(df1) <- colnames(df)
control <- trainControl(method = "cv", number=10)
nb <- train(imm_clus ~ ., data = df1, "naive_bayes", trainControl = control, tuneGrid = data.frame(usekernel = FALSE, laplace = 0, adjust=0))
```

# Apply trained canine model to human datasets

``` r
cellprop_allhuman <- rbind(cellprop_human[,ct],rbind(cellprop_human2[,ct],rbind(cellprop_human3[,ct],rbind(cellprop_human4[,ct], rbind(cellprop_human5[,ct], rbind(cellprop_human6[,ct], cellprop_human7[,ct]))))))

set.seed(2425)
km.res <- kmeans(cellprop_allhuman, centers = 3, nstart = 100, iter.max = 500)
imm_clus_all <- rep("ID", nrow(cellprop_allhuman))
imm_clus_all[km.res$cluster == 1] <- "IE"
imm_clus_all[km.res$cluster == 3] <- "IE-ECM"
imm_clus_all <- factor(imm_clus_all, levels = c("IE", "IE-ECM", "ID"))

human_test <- cbind(data.frame(cellprop_allhuman), imm_clus = imm_clus_all)
colnames(human_test) <- gsub("\\.","_", colnames(human_test))
preds <- predict(nb, human_test)
confusionMatrix(preds, human_test$imm_clus)
```

    ## Confusion Matrix and Statistics
    ## 
    ##           Reference
    ## Prediction  IE IE-ECM  ID
    ##     IE      42      0   0
    ##     IE-ECM  50    103  11
    ##     ID       0     91  97
    ## 
    ## Overall Statistics
    ##                                           
    ##                Accuracy : 0.6142          
    ##                  95% CI : (0.5642, 0.6625)
    ##     No Information Rate : 0.4924          
    ##     P-Value [Acc > NIR] : 7.796e-07       
    ##                                           
    ##                   Kappa : 0.3966          
    ##                                           
    ##  Mcnemar's Test P-Value : NA              
    ## 
    ## Statistics by Class:
    ## 
    ##                      Class: IE Class: IE-ECM Class: ID
    ## Sensitivity             0.4565        0.5309    0.8981
    ## Specificity             1.0000        0.6950    0.6818
    ## Pos Pred Value          1.0000        0.6280    0.5160
    ## Neg Pred Value          0.8580        0.6043    0.9466
    ## Prevalence              0.2335        0.4924    0.2741
    ## Detection Rate          0.1066        0.2614    0.2462
    ## Detection Prevalence    0.1066        0.4162    0.4772
    ## Balanced Accuracy       0.7283        0.6130    0.7900

``` r
sc_merged_subtypes <- predict(nb, cellprop_human_scOS[, colnames(human_test)[1:9]])
rownames(cellprop_human_scOS) <- colnames(bulk_exp_scOS)
Heatmap(cellprop_human_scOS, row_split = sc_merged_subtypes, cluster_columns = F, cluster_row_slices = F, row_gap = unit(1, "mm"), row_title_gp = gpar(col = c("#F8766D","#00BA38", "#619CFF")))
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-29-1.png)

``` r
names(sc_merged_subtypes) <- colnames(bulk_exp_scOS)
```

# Merge all human datasets

``` r
ds <- c(rep("TARGET", length(imm_clus_target)), rep("GSE21257", length(imm_clus2)), rep("GSE39055", length(imm_clus3)), rep("GSE33383",length(imm_clus4)), rep("GSE16091",length(imm_clus5)), rep("GSE30699",length(imm_clus6)), rep("GSE32981", length(imm_clus7)))

sample_type_bulk <- c(rep("Primary", length(imm_clus_target) + length(imm_clus2) + length(imm_clus3) + length(imm_clus4) + length(imm_clus5) + length(imm_clus6)), sample_type_7)
sample_type_all <- c(sample_type_bulk,sc_sample_type)
treatment_bulk <- c(rep("naive", length(imm_clus_target) + length(imm_clus2) + length(imm_clus3) + length(imm_clus4) + length(imm_clus5) + length(imm_clus6)), c("naive","post-chemo")[as.numeric(treatment_stat7 == "post-chemo")+1])
treatment <- c(treatment_bulk, rep("post-chemo",length(sc_merged_subtypes)))
platform = c(c("microarray","RNASeq")[as.numeric(ds == "TARGET")+1],rep("10x(scRNASeq)", length(sc_merged_subtypes)))
ds <- c(ds, rep("GSE152048", length(sc_merged_subtypes)))

ha = rowAnnotation(
    dataset = ds,
    platform = platform,
    sample = sample_type_all,
    treatment_status = treatment,
    col = list("dataset" = c("TARGET" = "#AF32CD", "GSE21257" =  "cyan","GSE39055" = "#A35711", "GSE33383" = "magenta", "GSE16091" = "#4500B2","GSE30699" = "#BCFF96", "GSE152048" = "#FB4232","GSE32981" = "#6775CD"),
               "platform" = c("microarray" = "#929292",
                              "RNASeq" = "#E2E0E7",
                              "10x(scRNASeq)" = "#3B3BFF"),
               "sample" = c("Primary" = "yellow", "Met" = "darkgreen", "Recurrent" = "darkorange"),
               "treatment_status" = c("naive" = "#EDAB32","post-chemo" = "#B30721"))
    
)
Heatmap(rbind(cellprop_allhuman,cellprop_human_scOS[,c(1,3:10)]), row_split = c(preds,sc_merged_subtypes), cluster_columns = F, cluster_row_slices = F, row_gap = unit(5, "mm"), row_title_gp = gpar(col = c("#00BA38", "#619CFF","#F8766D")), right_annotation = ha)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-30-1.png)

``` r
#chromosomal instability signature
names(imm_clus_all) <- c(names(imm_clus_target),names(imm_clus2), names(imm_clus3), names(imm_clus4))
CIN_1 <- CIN_signature(ee)
CIN_2 <- CIN_signature(expr_human2)
CIN_3 <- CIN_signature(expr_human3)
CIN_4 <- CIN_signature(expr_human4)
CIN_5 <- CIN_signature(expr_human5)
CIN_6 <- CIN_signature(expr_human6)
CIN_7 <- CIN_signature(expr_human7)
CIN_8 <- CIN_signature(bulk_exp_scOS)
CIN_all <- c(CIN_1,CIN_2,CIN_3,CIN_4, CIN_5, CIN_6, CIN_7, CIN_8)
names(CIN_all) <- c(names(imm_clus_all), names(sc_merged_subtypes))
p1 <- ggplot(data = data.frame(subtype = c(preds, sc_merged_subtypes), CIN_score = CIN_all), aes(x = subtype,y=CIN_score)) + geom_boxplot() +theme_bw() + stat_compare_means(method = "anova")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-30-2.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/CIN_human.pdf", width = 4, height = 4)


#type 1 interferon signature
type1_interferon_signature <- function(expr) {
  genes <- c("IFNA1", "IFNA2", "IFNA4", "IFNA5", "IFNA6", "IFNA7", "IFNA8", "IFNA10", "IFNA13", "IFNA14", "IFNA16", "IFNA17", "IFNA21", "IFNW1", "IFNE", "IFNK","IFNB1")
    mat <- as.data.frame(expr)
    mat <- mat[genes,]
    mat <- apply(as.matrix(mat),1,scale)
    scores <- rowMeans(mat, na.rm = T)
    return(scores)
}

IN_1 <- type1_interferon_signature(ee)
IN_2 <- type1_interferon_signature(expr_human2)
IN_3 <- type1_interferon_signature(expr_human3)
IN_4 <- type1_interferon_signature(expr_human4)
IN_5 <- type1_interferon_signature(expr_human5)
IN_6 <- type1_interferon_signature(expr_human6)
IN_7 <- type1_interferon_signature(expr_human7)
IN_8 <- type1_interferon_signature(bulk_exp_scOS)
IN_all <- c(IN_1,IN_2,IN_3,IN_4, IN_5, IN_6, IN_7, IN_8)
names(IN_all) <- c(names(imm_clus_all), names(sc_merged_subtypes))
p1 <- ggplot(data = data.frame(subtype = c(preds, sc_merged_subtypes), IN_score = IN_all), aes(x = subtype,y=IN_score)) + geom_boxplot() +theme_bw() + stat_compare_means( method = "anova")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-30-3.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/IN_human.pdf", width = 4, height = 4)


#Cytotoxic t cell signature (Rooney et al)
CYT_signature_human <- function(expr){
    genes <- c("GZMA","PRF1")
    mat <- as.data.frame(expr)
    mat <- (mat[genes,])
    scores <- sqrt(mat[1,]*mat[2,])
    return(scores)
}
CYT <- list()
CYT[[1]] <- CYT_signature_human(ee)
CYT[[2]] <- CYT_signature_human(expr_human2)
CYT[[3]] <- CYT_signature_human(expr_human3)
CYT[[4]] <- CYT_signature_human(expr_human4)
CYT[[5]] <- CYT_signature_human(expr_human5)
CYT[[6]] <- CYT_signature_human(expr_human6)
CYT[[7]] <- CYT_signature_human(expr_human7)
CYT[[8]] <- CYT_signature_human(bulk_exp_scOS)
CYT_all <- c()
for(i in 1: length(CYT)){
  cyt <-  as.numeric(CYT[[i]])
  names(cyt) <- colnames(CYT[[i]])
  CYT_all <- c(CYT_all,cyt)
}

p1 <- ggplot(data = data.frame(subtype = c(preds, sc_merged_subtypes), CYT_score = CYT_all), aes(x = subtype,y=CYT_score)) + geom_boxplot() +theme_bw() + stat_compare_means( method = "anova")
print(p1) + ylab("Cytotoxicity Score")
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-30-4.png)![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-30-5.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/CYT_human.pdf", width = 4, height = 4)


#expression of checkpoints
chkpoint_exp <- function(expr){
  out <- t(log2(data.frame(expr)[c("PDCD1","CD274","CTLA4","TIGIT","LAG3","HAVCR2"),]+1))
  colnames(out) <- c("PDCD1","CD274","CTLA4","TIGIT","LAG3","HAVCR2")
  return(data.frame(apply(out,2,scale)))
}

CP <- list()
CP[[1]] <- chkpoint_exp(ee)
CP[[2]] <- chkpoint_exp(expr_human2)
CP[[3]] <- chkpoint_exp(expr_human3)
CP[[4]] <- chkpoint_exp(expr_human4)
CP[[5]] <- chkpoint_exp(expr_human5)
CP[[6]] <- chkpoint_exp(expr_human6)
CP[[7]] <- chkpoint_exp(expr_human7)
CP[[8]] <- chkpoint_exp(bulk_exp_scOS)
CP_all <- Reduce(rbind, CP)

out <- data.frame(CP_all, subtype= c(preds, sc_merged_subtypes))
out <- out[!is.na(rowSums(out[,1:6])),]
dd_out <- melt(out, id.vars = "subtype")
colnames(dd_out) <- c("subtype","ICI_target","scaled_expression")
dd_out$ICI_target <- as.character(dd_out$ICI_target)
dd_out$subtype <- factor(dd_out$subtype, levels = c("IE","IE-ECM","ID"))
dd_out$ICI_target[dd_out$ICI_target == "PDCD1"] = "PD1"
dd_out$ICI_target[dd_out$ICI_target == "CD274"] = "PDL1"
dd_out$ICI_target[dd_out$ICI_target == "HAVCR2"] = "TIM3"
#p1 <- ggplot(dd_out, aes(x = subtype, y = log2TPM)) +
#  facet_wrap(~ICI_target, scales = "free_y") + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.signif") + theme_bw() + theme(text = element_text(size = 15)) + xlab("ICI target") + ylab("expression (log2(TPM)")

p1 <- ggboxplot(dd_out, x = "subtype", y="scaled_expression", facet.by = "ICI_target") + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means(method = "anova") + xlab("") + ylab("normalized expression")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-30-6.png)

``` r
#ggsave(plot = p1,filename = "~/Dog/human_immune_checkpoints.pdf", width = 6, height = 8)
```

# Evaluate progonostic power of TME subtype predictons using matched progression free survival outcome data

``` r
hc4 <- rbind(rbind(hc,hc2),rbind(hc3,rbind(hc44,rbind(hc5,rbind(hc6,hc7)))))
hc4$immune_subtype_1 <- imm_clus_all
hc4$ml_class <- preds
hc4$cyt <- CYT_all[names(imm_clus_all)]
hc4 <- hc4[sample_type_bulk == "Primary",]


surv <- survfit(Surv(OS_time, OS_status) ~ ml_class, data = hc4[!is.na(hc4$OS_time),])
diff <- survdiff(Surv(OS_time, OS_status) ~ ml_class, data = hc4[!is.na(hc4$OS_time),])

p1 <- ggsurvplot(surv, data = hc4[!is.na(hc4$OS_time),],
                  
                  legend.title = "",
                 legend = "right",
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                 xlim = c(0,4500),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                 palette = c("#882255","#CC6677","#88CCEE"),
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
#print(p1)
#ggsave(filename = "~/Dog/Human_OS_pred_primary.pdf", plot = p1$plot, width = 7, height = 4)


surv <- survfit(Surv(DFS_time, DFS_status) ~ ml_class, data = hc4[!is.na(hc4$DFS_time),])
diff <- survdiff(Surv(DFS_time, DFS_status) ~ ml_class, data = hc4[!is.na(hc4$DFS_time),])

p1 <- ggsurvplot(surv, data = hc4[!is.na(hc4$DFS_time),],

                  legend.title = "",
                 legend = "right",
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                 xlim = c(0,4500),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                 palette = c("#882255","#CC6677","#88CCEE"),
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-31-1.png)

``` r
#ggsave(filename = "~/Dog/Human_DFS_primary.pdf", plot = p1$plot, width = 7, height = 4)
#ggsave(p1$plot, filename = "~/Dog/aim3_human_DFS.pdf")
```

#plot proprtion of patients developing mets in 5 years for each TME
subtype (GSE33383)

``` r
pp <- as.character(preds)
names(pp)<- names(imm_clus_all)
dd_m <- as.matrix(table(pp[names(imm_clus4)][!is.na(GSE33383_pheno$`metastasis within 5yrs:ch1`)], GSE33383_pheno$`metastasis within 5yrs:ch1`[!is.na(GSE33383_pheno$`metastasis within 5yrs:ch1`)]))
dd_m <- (dd_m/rowSums(dd_m))*100
dd_m <- melt(dd_m)
```

    ## Warning in type.convert.default(X[[i]], ...): 'as.is' should be specified by
    ## the caller; using TRUE

    ## Warning in type.convert.default(X[[i]], ...): 'as.is' should be specified by
    ## the caller; using TRUE

``` r
colnames(dd_m) <- c("subtype","mets_within_5yrs","percentage")
dd_m$subtype <- factor(dd_m$subtype,levels = c("IE","IE-ECM","ID"))
p1 <- ggplot(as.data.frame(dd_m), aes(fill = mets_within_5yrs,
                      y = percentage, x = subtype))+
geom_bar(position = "fill", stat = "identity")+
ggtitle("Metastasis rates")+
theme(plot.title = element_text(hjust = 0.5)) + theme_bw()

print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-32-1.png)

``` r
#ggsave(filename = "~/Dog/Human_5yr_mets.pdf", plot = p1, width = 5, height = 4)


chisq.test(table(pp[names(imm_clus4)][!is.na(GSE33383_pheno$`metastasis within 5yrs:ch1`)], GSE33383_pheno$`metastasis within 5yrs:ch1`[!is.na(GSE33383_pheno$`metastasis within 5yrs:ch1`)]))
```

    ## Warning in
    ## chisq.test(table(pp[names(imm_clus4)][!is.na(GSE33383_pheno$`metastasis within
    ## 5yrs:ch1`)], : Chi-squared approximation may be incorrect

    ## 
    ##  Pearson's Chi-squared test
    ## 
    ## data:  table(pp[names(imm_clus4)][!is.na(GSE33383_pheno$`metastasis within 5yrs:ch1`)],     GSE33383_pheno$`metastasis within 5yrs:ch1`[!is.na(GSE33383_pheno$`metastasis within 5yrs:ch1`)])
    ## X-squared = 7.7036, df = 2, p-value = 0.02124

# Evaluate prognostic power of cytolytic sores

``` r
hc4$cyt_score <- c("low","high")[as.numeric(hc4$cyt > median(hc4$cyt, na.rm = TRUE))+1]
surv <- survfit(Surv(OS_time, OS_status) ~ cyt_score, data = hc4[!is.na(hc4$OS_time),])
diff <- survdiff(Surv(OS_time, OS_status) ~ cyt_score, data = hc4[!is.na(hc4$OS_time),])

p1 <- ggsurvplot(surv, data = hc4[!is.na(hc4$OS_time),],
                  
                  legend.title = "",
                 legend = "right",
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                 xlim = c(0,4500),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                 palette = c("#882255","#88CCEE","#CC6677"),
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
#print(p1)
#ggsave(filename = "~/Dog/Human_OS_pred_primary.pdf", plot = p1$plot, width = 7, height = 4)


surv <- survfit(Surv(DFS_time, DFS_status) ~ cyt_score, data = hc4[!is.na(hc4$DFS_time),])
diff <- survdiff(Surv(DFS_time, DFS_status) ~ cyt_score, data = hc4[!is.na(hc4$DFS_time),])

p1 <- ggsurvplot(surv, data = hc4[!is.na(hc4$DFS_time),],

                  legend.title = "",
                 legend = "right",
                  conf.int = F,
                  pval = TRUE,
                  risk.table = TRUE,
                  tables.height = 0.2,
                  tables.theme = theme_cleantable(),
                  risk.table.y.text = FALSE,
                 pval.coord = c(0, 0.03),
                 xlim = c(0,4500),
                  # Color palettes. Use custom color: c("#E7B800", "#2E9FDF"),
                  # or brewer color (e.g.: "Dark2"), or ggsci color (e.g.: "jco")
                 palette = c("#882255","#88CCEE", "#CC6677"),
                  ggtheme = theme(text = element_text(size = 15)) +theme_bw() # Change ggplot2 theme
) + xlab("Time (days)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-33-1.png)

# Run multivariate cox proportional hazards analysis to account for confounding effects

``` r
cph_dat <- data.frame(time = hc4$DFS_time,
subtype = hc4$ml_class,
cyt_score = hc4$cyt,
status = hc4$DFS_status,
stage = hc4$met_at_daig,
gender = hc4$gender,
age = scale(hc4$age, center = T, scale = T), necrosis = hc4$necrosis)
cph_dat <- cph_dat[!is.na(cph_dat$necrosis) & !is.na(hc4$DFS_time),]
cph_dat$subtype <- relevel(as.factor(cph_dat$subtype), ref = "ID")
gender <- relevel(as.factor(cph_dat$gender), ref = "M")
cph_dat$stage[cph_dat$stage == "NON-METASTATIC"] <- "LOCALIZED"
cph_dat$stage <- relevel(as.factor(cph_dat$stage), ref = "LOCALIZED")
cph_dat$predictor <- rep(3, nrow(cph_dat))
cph_dat$predictor[cph_dat$subtype == "IE-ECM"] <- 2
cph_dat$predictor[cph_dat$subtype == "IE"] <- 1

fit <- coxph(Surv(time, status) ~ age + gender + stage + necrosis + subtype, data = cph_dat)
p2 <-ggforest(fit, data = cph_dat,fontsize = 0.7, noDigits = 2) + theme(text = element_text(size = 25))
print(p2)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-34-1.png)

``` r
#ggsave(plot = p2, filename = "~/Dog/human_cph_results_cyt.pdf", width = 5, height = 6)
```

# plot distribution of mcp counter scores for both canine and human samples

``` r
dd_rnaseq <- data.frame(cellprop_rnaseq[,c(1:10)], subtype = imm_clus)
dd_rnaseq <- melt(dd_rnaseq, id.vars = "subtype")
p1 <- ggboxplot(dd_rnaseq, x = "subtype", y="value", facet.by = "variable",) + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means( method = "anova") + xlab("") + ylab("MCP counter score (normalized)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-35-1.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/caninernaseq_mcpcounter_boxplots.pdf", width = 14, height = 9)

dd_rnaseq <- data.frame(cellprop_nano[,c(1:10)], subtype = imm_clus_nano)
dd_rnaseq <- melt(dd_rnaseq, id.vars = "subtype")
p1 <- ggboxplot(dd_rnaseq, x = "subtype", y="value", facet.by = "variable") + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means(method = "anova") + xlab("") + ylab("MCP counter score (normalized)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-35-2.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/caninenano_mcpcounter_boxplots.pdf", width = 14, height = 9)

dd_rnaseq <- data.frame(rbind(cellprop_rnaseq[,c(1,3:10)],cellprop_nano[,c(1,3:10)]), subtype = c(imm_clus,imm_clus_nano))
dd_rnaseq <- melt(dd_rnaseq, id.vars = "subtype")
p1 <- ggboxplot(dd_rnaseq, x = "subtype", y="value", facet.by = "variable") + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means(method = "anova") + xlab("") + ylab("MCP counter score (normalized)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-35-3.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/allcanine_mcpcounter_boxplots.pdf", width = 11, height = 9)
dd_rnaseq <- data.frame(rbind(cellprop_allhuman), subtype = preds)
dd_rnaseq <- melt(dd_rnaseq, id.vars = "subtype")
p1 <- ggboxplot(dd_rnaseq, x = "subtype", y="value", facet.by = "variable") + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means(method = "anova") + xlab("") + ylab("MCP counter score (normalized)")
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-35-4.png)

``` r
#ggsave(plot = p1, filename = "~/Dog/allhuman_mcpcounter_boxplots.pdf", width = 11, height = 9)
```

# read single cell celltype fractions from Zhou et al, Nat Comm 2020 and compare against predicted TME subtypes

``` r
human_OS_sc_celltypes <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S5a", skip = 2)
```

    ## New names:
    ## • `` -> `...1`

``` r
human_OS_sc_celltypes$...1 <- sapply(human_OS_sc_celltypes$...1, function(x) strsplit(x,split = "\\.")[[1]][2])
human_OS_sc_celltypes <- human_OS_sc_celltypes[!is.na(human_OS_sc_celltypes$...1),]
names(human_OS_sc_celltypes$...1) <- NULL
human_OS_sc_celltypes <- as.data.frame(human_OS_sc_celltypes)
rownames(human_OS_sc_celltypes) <- human_OS_sc_celltypes$...1
human_OS_sc_celltypes <- human_OS_sc_celltypes[,-1]
human_OS_sc_celltypes <- as.data.frame(t(human_OS_sc_celltypes))

scOS_cancercell <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S6b", skip = 2)
```

    ## New names:
    ## • `` -> `...1`

``` r
scOS_cancercell <- as.data.frame(scOS_cancercell)
rownames(scOS_cancercell) <- scOS_cancercell$...1
scOS_cancercell <- scOS_cancercell[,-1]
scOS_cancercell <- as.data.frame(scOS_cancercell[rownames(human_OS_sc_celltypes),])


scOS_chondro <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S6g", skip = 1)
scOS_chondro <- as.data.frame(scOS_chondro)
rownames(scOS_chondro) <- scOS_chondro$Patient
scOS_chondro <- scOS_chondro[,-1]
scOS_chondro <- scOS_chondro[,-5]
scOS_chondro <- as.data.frame(scOS_chondro[rownames(human_OS_sc_celltypes),])


scOS_osteoclast <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S11c", skip = 1)
scOS_osteoclast <- as.data.frame(scOS_osteoclast)
rownames(scOS_osteoclast) <- scOS_osteoclast$`Cell type`
scOS_osteoclast <- scOS_osteoclast[,-1]
scOS_osteoclast <- t(scOS_osteoclast)
scOS_osteoclast <- as.data.frame(scOS_osteoclast[rownames(human_OS_sc_celltypes),])

scOS_MSC <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S14a", skip = 1)
scOS_MSC <- as.data.frame(scOS_MSC)
rownames(scOS_MSC) <- scOS_MSC$`Cell type`
scOS_MSC<- scOS_MSC[,-1]
scOS_MSC <- t(scOS_MSC)
scOS_MSC <- as.data.frame(scOS_MSC[rownames(human_OS_sc_celltypes),])

scOS_CAF <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S14d", skip = 1)
scOS_CAF <- as.data.frame(scOS_CAF)
rownames(scOS_CAF) <- scOS_CAF$`Cell type`
scOS_CAF <- scOS_CAF[,-1]
scOS_CAF <- t(scOS_CAF)
scOS_CAF <- as.data.frame(scOS_CAF[rownames(human_OS_sc_celltypes),])

scOS_Myeloid <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S15b", skip = 1)
scOS_Myeloid <- as.data.frame(scOS_Myeloid)
rownames(scOS_Myeloid) <- scOS_Myeloid$`Cell type`
scOS_Myeloid <- scOS_Myeloid[,-1]
scOS_Myeloid <- t(scOS_Myeloid)
scOS_Myeloid <- as.data.frame(scOS_Myeloid[rownames(human_OS_sc_celltypes),])


scOS_TIL <- read_excel("~/Dog/sourcedat_human_OS_singlecell.xlsx", sheet = "Figure S16b", skip = 1)
scOS_TIL <- as.data.frame(scOS_TIL)
rownames(scOS_TIL) <- scOS_TIL$Patient
scOS_TIL <- scOS_TIL[,-1]
scOS_TIL <- as.data.frame(scOS_TIL[rownames(human_OS_sc_celltypes),])


total <- rowSums(human_OS_sc_celltypes)
scOS_cancercell$Osteoblastic_1 <- scOS_cancercell$Osteoblastic_1/total
scOS_cancercell$Osteoblastic_2 <- scOS_cancercell$Osteoblastic_2/total
scOS_cancercell$Osteoblastic_3 <- scOS_cancercell$Osteoblastic_3/total
scOS_cancercell$Osteoblastic_4 <- scOS_cancercell$Osteoblastic_4/total
scOS_cancercell$Osteoblastic_5 <- scOS_cancercell$Osteoblastic_5/total
scOS_cancercell$Osteoblastic_6 <- scOS_cancercell$Osteoblastic_6/total

scOS_cancercell$`Chondroblastic OS` <- scOS_cancercell$`Chondroblastic OS`/total
scOS_chondro$Chondro_Proli <- scOS_chondro$Chondro_Proli/total
scOS_chondro$Chondro_hyper_1 <- scOS_chondro$Chondro_hyper_1/total
scOS_chondro$Chondro_hyper_2 <- scOS_chondro$Chondro_hyper_2/total
scOS_chondro$Chondro_trans_diff <- scOS_chondro$Chondro_trans_diff/total

scOS_osteoclast$OC_progenitor <- scOS_osteoclast$OC_progenitor/total
scOS_osteoclast$OC_immature <- scOS_osteoclast$OC_immature/total
scOS_osteoclast$OC_mature <- scOS_osteoclast$OC_mature/total

scOS_CAF$`COL14A1+ fibroblast` <- scOS_CAF$`COL14A1+ fibroblast` / total
scOS_CAF$`Smooth muscle cell` <- scOS_CAF$`Smooth muscle cell` / total
scOS_CAF$Myofibroblast <- scOS_CAF$Myofibroblast / total

scOS_MSC$`NT5E+MSC` <- scOS_MSC$`NT5E+MSC` / total
scOS_MSC$`WISP2+MSC` <- scOS_MSC$`WISP2+MSC` / total
scOS_MSC$`CLEC11A+MSC` <- scOS_MSC$`CLEC11A+MSC` / total

scOS_Myeloid$`CD14+_monocytes` <- scOS_Myeloid$`CD14+_monocytes` / total
scOS_Myeloid$`CXCL8+_monocytes` <- scOS_Myeloid$`CXCL8+_monocytes` / total
scOS_Myeloid$M2_TAM <- scOS_Myeloid$M2_TAM / total
scOS_Myeloid$M1_macrophage <- scOS_Myeloid$M1_macrophage / total
scOS_Myeloid$`FABP4+_macrophage` <- scOS_Myeloid$`FABP4+_macrophage` / total
scOS_Myeloid$Neutrophil <- scOS_Myeloid$Neutrophil / total
scOS_Myeloid$`CD14+/CD163+_DC` <- scOS_Myeloid$`CD14+/CD163+_DC` / total
scOS_Myeloid$`CD1C+_DC` <- scOS_Myeloid$`CD1C+_DC` / total
scOS_Myeloid$`CD141+/CLEC9A+_DC` <- scOS_Myeloid$`CD141+/CLEC9A+_DC` / total

scOS_TIL$`CD4+T` <- scOS_TIL$`CD4+T` / total
scOS_TIL$`CD4-/CD8-` <- scOS_TIL$`CD4-/CD8-` / total
scOS_TIL$`T-reg` <- scOS_TIL$`T-reg` / total
scOS_TIL$`CXCL13+T` <- scOS_TIL$`CXCL13+T` / total
scOS_TIL$`CD8+T` <- scOS_TIL$`CD8+T` / total
scOS_TIL$NKT <- scOS_TIL$NKT / total
scOS_TIL$NK <- scOS_TIL$NK / total
scOS_TIL$B_cell <- scOS_TIL$B_cell / total
scOS_TIL$`Proliferating T` <- scOS_TIL$`Proliferating T` / total


scOS_detailed <- list()
scOS_detailed[[1]] <- scOS_cancercell
scOS_detailed[[2]] <- scOS_chondro
scOS_detailed[[3]] <- scOS_osteoclast
scOS_detailed[[4]] <- scOS_TIL
scOS_detailed[[5]] <- scOS_Myeloid
scOS_detailed[[6]] <- scOS_CAF
scOS_detailed[[7]] <- data.frame(Pericytes = human_OS_sc_celltypes$Pericytes) / total
scOS_detailed[[8]] <- scOS_MSC
scOS_detailed[[9]] <- data.frame(Myoblast = human_OS_sc_celltypes$Myoblast) / total
scOS_detailed[[10]] <- data.frame(Endothelial = human_OS_sc_celltypes$Endothelial) / total

scOS_detailed_all <- cbind(scOS_detailed[[1]], scOS_detailed[[2]],
                           scOS_detailed[[3]], scOS_detailed[[4]],
                           scOS_detailed[[5]], scOS_detailed[[6]],
                           scOS_detailed[[7]], scOS_detailed[[8]],
                           scOS_detailed[[9]], scOS_detailed[[10]])

#plot results
ha = rowAnnotation(
    sample = sc_sample_type,
    subtype = sc_merged_subtypes,
    col = list("sample" = c("Primary" = "yellow", "Met" =  "darkgreen", "Recurrent" = "orange"),
               "subtype" = c("IE" = "#882255", "IE-ECM" =  "#CC6677", "ID" = "#88CCEE")
    )
)

aa <- apply(scOS_detailed_all[names(sc_merged_subtypes),],2,scale)
rownames(aa) <- names(sc_merged_subtypes)

p1 <- Heatmap(aa, row_split = sc_merged_subtypes, cluster_columns = F, cluster_row_slices = F, cluster_rows = F, row_gap = unit(1, "mm"), row_title_gp = gpar(col = c("#F8766D","#00BA38", "#619CFF")), right_annotation = ha)
print(p1)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-36-1.png)

``` r
#pdf("~/Dog/scOS_heatmap_extended.pdf", width = 12, height = 7)
#print(p1)
#dev.off()



aa <- human_OS_sc_celltypes[names(sc_merged_subtypes),]
aa <- human_OS_sc_celltypes[names(sc_merged_subtypes),]/rowSums(human_OS_sc_celltypes[names(sc_merged_subtypes),])
aa <- apply(aa,2,scale)
rownames(aa) <- names(sc_merged_subtypes)
Heatmap(aa, row_split = sc_merged_subtypes, cluster_columns = F, cluster_row_slices = F, cluster_rows = F,row_gap = unit(1, "mm"), row_title_gp = gpar(col = c("#F8766D","#00BA38", "#619CFF")), right_annotation = ha)
```

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-36-2.png)

``` r
#pdf("~/Dog/scOS_heatmap.pdf")
#Heatmap(aa, row_split = sc_merged_subtypes, cluster_columns = F, cluster_row_slices = F, cluster_rows = F,row_gap = unit(1, "mm"), row_title_gp = gpar(col = c("#F8766D","#00BA38", "#619CFF")), right_annotation = ha)
#dev.off()

dd_rnaseq <- data.frame(scOS_detailed_all[names(sc_merged_subtypes),], subtype = sc_merged_subtypes)
dd_rnaseq <- melt(dd_rnaseq, id.vars = "subtype")
p1 <- ggboxplot(dd_rnaseq, x = "subtype", y="value", facet.by = "variable") + theme_bw() + theme(text = element_text(size = 15)) + stat_compare_means(comparisons = list(c("IE","IE-ECM"),c("IE-ECM","ID"),c("IE","ID")), method = "wilcox", label = "p.signif") + xlab("") + ylab("proportions")
print(p1)
```

    ## Warning in wilcox.test.default(0, c(0, 0, 0.000821900486817981, 0,
    ## 6.94251596778673e-05, : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.00011887779362815, 0.00690483338336836, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0, 0, 0.0832858607724735, 0), paired =
    ## FALSE): cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0, 0, 0, 0, 0, 0), paired = FALSE): cannot
    ## compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0, 0, 0.0832858607724735, 0), c(0, 0, 0, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0.000356633380884451, 0, 0.191409365386806,
    ## : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0, 0.00137589433131536,
    ## 0.000632231143706139, : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.000356633380884451, 0, 0.191409365386806, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0.152477763659466, c(0.0306704707560628, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.0306704707560628, 0.0054037826478535, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0.00381194409148666, c(0.00855920114122682, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.00855920114122682, 0, 0, 0.003397341211226:
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0.000713266761768902, 0, 0, 0), paired =
    ## FALSE): cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.000713266761768902, 0, 0, 0),
    ## c(0.000755144421370587, : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0.0986022871664549, c(0, 0, 0, 0, 0,
    ## 0.00034965034965035: cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.00546837850689491, 0, 0.000455736584254301,
    ## : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0.000762388818297332, c(0.00011887779362815, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.00011887779362815, 0, 0.000227868292127151,
    ## : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(1.41319293865906e-06, 0.000684958606424497,
    ## : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0, 0, 0, 0, 0, 0), paired = FALSE): cannot
    ## compute exact p-value with ties

    ## Warning in wilcox.test.default(c(1.41319293865906e-06, 0.000684958606424497, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0.000188786105342647, 0,
    ## 0.000189669343111842, : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0.000356633380884451, 0.000300210147102972, :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0, 0.000300210147102972, 0.00478523413467016,
    ## : cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0, 0.032422695887121, 0,
    ## 0.000295420974889217: cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(0, c(0, 0, 0.000126446228741228, 0, 0, 0), :
    ## cannot compute exact p-value with ties

    ## Warning in wilcox.test.default(c(0, 0.032422695887121, 0, 0.000295420974889217:
    ## cannot compute exact p-value with ties

![](comparative_deconv_analysis_files/figure-markdown_github/unnamed-chunk-36-3.png)
