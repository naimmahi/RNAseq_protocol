# The Cancer Genome Atlas - paired samples

The data provided below is taken from [TCGA](http://cancergenome.nih.gov/). It contains the RNA-Seq V2 data (RSEM, gene level, raw) for all publicly available data sets (April 2016). In total there would be around 11'700 samples. However, many of them were not sampled in a "paired fashion" (for most patients/participants there is only data on the tumor sample available). The data below is restricted to the paired cases - meaning that we have both, normal and tumor tissue samples, for one study participant (this are almost exclusively sample types "NT" and "PT" - I therefore restricted it further to those two; see [TCGA-Barcode](https://wiki.nci.nih.gov/display/TCGA/Working+with+TCGA+Data#WebServices-Barcode-UUIDMapping) for more explanations). The sample annotation table is based on the TCGA barcodes. Note that I added "uuid_" in front of the UUIDs. The reason for this is that they frequently start with a number, which does not work if you want to use them as colnames() in R. 

Download [the archive with the data and the annotation](TCGA_pairedData.csv?raw=true) and unpack it. Place the file into the working directory specified in the script.

## Identify genes differentially expressed between normal and tumor tissues - for each cancer type

Warning: The script requires quite some RAM. You should have at least 24 GB.

```R
library("RNAseqWrapper")

# choose a working directory
rDir <- "/path/to/your/working/directory"

# load the data
sampleTab <- read.csv(file.path(rDir, "pairedSamplesAnnotation.csv"), colClasses = "character")
myData <- read.csv(file.path(rDir, "pairedSamplesRaw.csv"), row.names = 1)

# do one pairwise comparison per cancer type - normal vs tumor
# use the patient/participant ID as batch (treat it as paired)
# we are now only using edgeR with trended dispersion estimates.
# Feel free to try something else.
pairwiseResults <- list()
for (CSS in unique(sampleTab$dataSet)) {
  cat(CSS, "\n")
  subSampleTab <- subset(sampleTab, dataSet == CSS)
  if (sum(subSampleTab$sampleType == "NT") < 3) { next }
  subData <- myData[,subSampleTab$uuid]
  pairwiseResults[[CSS]] <- f.two.groups.edgeR(subData, subSampleTab$sampleType,
                                               c("NT", "TP"), method = "trended", 
                                               batch = subSampleTab$participant)
  # the logFCs correspond now to tumor-normal
}

# write out the results for each comparison
for (CSS in names(pairwiseResults)) {
  cat(CSS, "\n")
  deTab <- pairwiseResults[[CSS]]$get_table()
  write.csv(deTab, file.path(rDir, paste0("PWWB_", CSS, ".csv")))
}

#########################################################################################
# you might be interested in a specific gene - extract the results for it
myGeneOfInterest <- "<entrezSymbol>|<entrezID>"
singleGeneResults <- list()
for (CSS in names(pairwiseResults)) {
  deTab <- pairwiseResults[[CSS]]$get_table()
  singleGeneResults[[CSS]] <- c(CSS, deTab[myGeneOfInterest,], 
                                length(unique(subset(sampleTab, dataSet == CSS)$participant)),
                                superSetDesc[unlist(strsplit(CSS, ".", TRUE))[1], "disease"])
}
singleGeneResults <- do.call("rbind", singleGeneResults)
colnames(singleGeneResults) <- c("cancerType", "logCPM", "logFC", "pVal", "adjP", "numParticipants", "cancerDescription")
write.csv(singleGeneResults, file.path(rDir, "singleGeneResult.csv"), row.names = FALSE)

#########################################################################################
# you could also try to get the genes which are consistently enriched in either normal or tumor tissue
# across several different types of cancer.
sigCollection <- list(tumor = list(), normal = list())
for (CSS in names(pairwiseResults)) {
  #if(length(unique(subset(sampleTab, dataSet == CSS)$participant)) < 30) {next}
  temp <- pairwiseResults[[CSS]]$get_significant_entries(1, 1e6, 2)
  sigCollection$tumor[[CSS]] <- temp$TP
  sigCollection$normal[[CSS]] <- temp$NT
}
anyTumor <- Reduce(union, sigCollection$tumor)
anyNormal <- Reduce(union, sigCollection$normal)
deMatTumor <- matrix(0, nrow = length(anyTumor), ncol = length(pairwiseResults), dimnames = list(anyTumor, names(pairwiseResults)))
deMatNormal <- matrix(0, nrow = length(anyNormal), ncol = length(pairwiseResults), dimnames = list(anyNormal, names(pairwiseResults)))
for (CSS in names(pairwiseResults)) {
  #if(length(unique(subset(sampleTab, dataSet == CSS)$participant)) < 30) {next}
  temp <- pairwiseResults[[CSS]]$get_significant_entries(1, 1e6, 2) # FDR < 0.000001 and logFC > 2
  deMatTumor[temp$TP, CSS] <- 1
  deMatNormal[temp$NT, CSS] <- 1
}
deCountsTumor <- sort(apply(deMatTumor, 1, sum), decreasing = TRUE) # counts how often a gene was significantly upregulated in tumor tissue
deCountsNormal <- sort(apply(deMatNormal, 1, sum), decreasing = TRUE) # counts how often a gene was significantly upregulated in normal tissue
head(deCountsTumor) # this shows the genes which were most often found to be upregulated in tumor tissue
head(deCountsNormal) # this shows the genes which were most often found to be upregulated in normal tissue
sum(deCountsTumor > 10) # 325
sum(deCountsNormal > 10) # 341
```

Hm - silly enough, the alcohol dehydrogenase 1B is one of the top-enriched genes in normal tissues :)