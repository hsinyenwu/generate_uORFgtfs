## Here we describe two ways to generate gtf files for uORFs

Remember there could be multiple uORFs for one 5'UTR, and there will be multiple gtf files generate in that situation.

The uORF gtf files generated here could be used as the input gtf file for the `uorf.structure` function (RiboPlotR).

**Please change the path/directory if you want to try out the code below.**

### 1. Generate gtf files for uORFs using genomic sequences and annotation for 5'UTRs and transcripts   

Note:
1. Each uORF gtf generated by the `generate_uORFgtfs` function only include the transcript and the CDS range(s) for one uORF. 
2. Here we only consider uORFs start with AUG but you can modify the start sequences in the findUORFs (see ORFik package for detail) function.
3. Requirements: (1) a fasta file for genome sequence (2) a gtf files includes the range information for exon and main ORF (CDS) ranges. 
4. The following code works in Mac OS and Linux, but might require slight modification for Windows system.
5. The compressed files for: TAIR10_chr_1.fa and Araport11+CTRL_20181206.gtf are attached so you can try this function out.

```R
# Make uORF GTFs using genomic sequences and annotation for 5'UTRs and transcripts for one transcript
# rm(list=ls())
library(GenomicRanges)
library(GenomicFeatures)
library(Rsamtools)
library(ORFik)
library(rtracklayer)

# Load the genome sequence file (e.g. TAIR10 genome sequence):
FA <- FaFile("~/Desktop/TAIR10_chr_1.fa")
# Make the TxDb file (Use a gtf from Araport11)
txdb <- makeTxDbFromGFF("~/Desktop/Araport11+CTRL_20181206.gtf",format="gtf", dataSource="Araport11",organism="Arabidopsis")
# Extract 5'UTR ranges
fiveUTR <- fiveUTRsByTranscript(txdb, use.names=T)
# Extract CDS ranges
CDS <- cdsBy(txdb,by="tx", use.names=T)
# Extract exon ranges
exonRanges <- exonsBy(txdb,by="tx", use.names=T)

# generate_uORFgtfs function
# x is the gene name
# y is the transcript name
# z is the path for the new gtf files

generate_uORFgtfs <-function(x,y,z){
  #Find all possible uORFs
  uORF <- findUORFs(fiveUTRs=fiveUTR[y], startCodon="ATG", fa=FA, longestORF=T, cds=CDS[y], restrictUpstreamToTx=T)
  #Export the exon info for the uORF containing transcript to a temp gtf file
  export.gff(exonRanges[y],con=paste0(z, "uORF_exon_ranges.gtf"), format="gff2")
  #Read in the exon info. Now it is in the gtf format
  EXONu <- read.delim(paste0(z,"uORF_exon_ranges.gtf"), skip=4, header=F)
  #Modify the column 3 and 9
  EXONu$V3 <- "mRNA"
  EXONu$V9 <- paste0("gene_id ","\"",x,"\"; ", "transcript_id ","\"",y,"\";")
  
  for (i in seq_along(uORF)) {
  #Export the uORF CDS range for uORF i (i is the number of a specific uORF)
    export.gff(uORF[i], con=paste0(z,"uORF_CDS_ranges.gtf"), format="gff2")
    #Read in the CDS range
    CDSu <- read.delim(paste0(z,"uORF_CDS_ranges.gtf"), skip=4, header=F)
    #Modify column 3 and 9
    CDSu$V3 <- "CDS"
    CDSu$V9 <- paste0("gene_id ", "\"",x, "\"; ", "transcript_id ","\"",y,"\";")
    rbind(EXONu,CDSu)
    write.table(rbind(EXONu,CDSu), paste0(z,y,".",i,".gtf"), quote = F, col.names = F, row.names = F, sep = "\t",append = F)
  }
}

#Example:
generate_uORFgtfs(x="AT1G01060",y="AT1G01060.1",z="~/Desktop/test/") #Should generate five uORF gtf files.

```


### 2. Generate the gtf files for RiboTaper defined uORFs

```R
# rm(list=ls())
library(GenomicRanges)
library(GenomicFeatures)
library(Rsamtools)
library(ORFik)
library(rtracklayer)

# Load genome sequences
FA <- FaFile("~/Desktop/Leaky_scanning/TAIR10_chr_1.fas")
# Generate txdb object
txdb <- makeTxDbFromGFF("~/Desktop/CTRL_v1/Araport11+CTRL_20181206.gtf",format="gtf", dataSource="Araport11",organism="Arabidopsis")
# Load GTF file
GTF <- read.delim("~/Desktop/CTRL_v1/Araport11+CTRL_20181206.gtf",header=F,stringsAsFactors = F)
# Generate 5'UTR ranges
fiveUTR <- fiveUTRsByTranscript(txdb, use.names=T)
# Generate CDS ranges
CDS <- cdsBy(txdb,by="tx", use.names=T)

#load RiboTaper output ORFs_max_filt file
Exp <- read.delim(file="~/Desktop/uORF_gtfs/ORFs_max_filt_example.tsv",header=T,stringsAsFactors=F,sep="\t") #file included 
Exp_uORFs <- Exp %>% filter(category=="uORF") %>% mutate(ORF_pept_length=nchar(ORF_pept))
nrow(Exp_uORFs)

for(i in 1:nrow(Exp_uORFs)){
  if(i%%100==0) print(i) 
  # a is the 5'UTR range for the ith uORF
  a=fiveUTR[Exp_uORFs$transcript_id[i]]
  # b is the CDS range for the ith uORF
  b=CDS[Exp_uORFs$transcript_id[i]]
  # d is the output ranges for the findUORFs function
  d=findUORFs(fiveUTRs=a,fa=FA, startCodon="ATG",longestORF=F,cds=b,restrictUpstreamToTx=T)
  # then identify the RiboTaper identified uORFs that is identical with which findUORFs defined uORFs 
  Pept <- as.character(translate(extractTranscriptSeqs(FA,d)))
  Pept_remove_stop <- gsub("\\*","",Pept)
  Pept_uORF <- d[which(Pept_remove_stop==Exp_uORFs[i,]$ORF_pept)]
  # Make the mRNA rows for gtf file
  mRNAr = GTF %>% filter(V3=="mRNA",grepl(pattern=Exp_uORFs$transcript_id[i],V9))
  # Make the CDS rows for gtf file
  CDSr <- export.gff(Pept_uORF,con=paste0("~/Desktop/uORF_gtfs/cds.gtf"),format="gff2")
  CDSr <- read.delim("~/Desktop/uORF_gtfs/cds.gtf",skip=4,header=F)
  CDSr$V3 <- "CDS"
  CDSr$V9 <- paste0("gene_id ","\"",substr(Exp_uORFs$transcript_id[i],1,9),"\"; ", "transcript_id ","\"",Exp_uORFs$transcript_id[i],"\";")
  #output the file
  write.table(rbind(mRNAr,CDSr),paste0("~/Desktop/uORF_gtfs/",Exp_uORFs$ORF_id_tr[i],".gtf"),quote = F, col.names = F, row.names = F, sep = "\t",append = F)
}
```

Citation: Please cite our paper: https://plantmethods.biomedcentral.com/articles/10.1186/s13007-021-00824-4 if you use information provided here for your research. Thanks!
