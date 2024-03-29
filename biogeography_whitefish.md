Biogeography of European whitefish
================
Marco Crotti
28 April, 2021

  - [Pipeline for UK whitefish biogeography project using
    ddRADseq](#pipeline-for-uk-whitefish-biogeography-project-using-ddradseq)
      - [Set up working environment](#set-up-working-environment)
      - [Preparing the data for population genomics
        analyses](#preparing-the-data-for-population-genomics-analyses)
          - [01. Demultiplex raw reads](#demultiplex-raw-reads)
          - [02. <span>Trimmomatic</span>
            filtering](#trimmomatic-filtering)
          - [03. Align to European whitefish genome
            assembly](#align-to-european-whitefish-genome-assembly)
          - [04. Build Stacks catalog](#build-stacks-catalog)
      - [Population genomics and phylogenetics
        analyses](#population-genomics-and-phylogenetics-analyses)
          - [Generate a vcf file using
            <span>populations</span>](#generate-a-vcf-file-using-populations)
              - [Filter the vcf file](#filter-the-vcf-file)
          - [Genetic diversity and allelic
            richness](#genetic-diversity-and-allelic-richness)
              - [Genetic diversity](#genetic-diversity)
              - [Allelic richness](#allelic-richness)
          - [Convert the vcf file to
            phylip](#convert-the-vcf-file-to-phylip)
          - [Admxiture analysis](#admxiture-analysis)
          - [DAPC analysis in adegenet](#dapc-analysis-in-adegenet)
          - [fineRADstructure analysis](#fineradstructure-analysis)
          - [Principal component analysis using
            SNPRelate](#principal-component-analysis-using-snprelate)
          - [Genome-wide FST](#genome-wide-fst)
          - [TreeMix](#treemix)

## Pipeline for UK whitefish biogeography project using ddRADseq

### Set up working environment

Let’s create the folders that are going to contain the output data from
all the pipeline components.

``` bash
mkdir ./Desktop/biogeography
mkdir ./Desktop/biogeography/00.Raw_reads ./Desktop/biogeography/01.Demultiplexed_reads ./Desktop/biogeography/02.Trimmomatic_filtering ./Desktop/biogeography/03.Assembly ./Desktop/biogeography/04.bam_alignments ./Desktop/biogeography/05.Stacks ./Desktop/biogeography/06.Phylogenetics ./Desktop/biogeography/07.Population_genetics
```

### Preparing the data for population genomics analyses

We are going to demultiplex raw reads, filter low reads out, align to
reference genome, and build a STACKS catalog.

#### 01\. Demultiplex raw reads

Use
[process\_radtags](http://catchenlab.life.illinois.edu/stacks/comp/process_radtags.php)
to demultiplex Illumina raw data.

``` bash
process_radtags -P -c -q -r -p ./00.Raw_reads/ -o ./01.Demultiplexed_reads -b ./biogeography_barcodes.txt --inline_inline -i gzfastq -y gzfastq --renz_1 pstI --renz_2 mspI -t 65
```

#### 02\. [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) filtering

Remove the first 5 bp and 3 bp from the forward and reverse reads to
remove the enzyme cut site, single end reads.

``` bash
# Forward reads
for infile in ./01.Demultiplexed_reads/*.1.fq.gz
do
base=$(basename $infile .fq.gz)
java -jar /usr/local/bin/trimmomatic-0.38.jar SE -threads 4 $infile ./02.Trimmomatic_filtering/$base.fq.gz HEADCROP:5
done

# Reverse reads
for infile in ./01.Demultiplexed_reads/*.2.fq.gz
do
base=$(basename $infile .fq.gz)
java -jar /usr/local/bin/trimmomatic-0.38.jar SE -threads 4 $infile ./02.Trimmomatic_filtering/$base.fq.gz HEADCROP:3
done
```

Do paired-end filtering.

``` bash
for R1 in *.1.fq.gz
do
R2=${R1//1.fq.gz/2.fq.gz}
R1paired=${R1//.1.fq.gz/.P1.fq.gz}
R1unpaired=${R1//.1.fq.gz/.U1.fq.gz}    
R2paired=${R2//.2.fq.gz/.P2.fq.gz}
R2unpaired=${R2//.2.fq.gz/.U2.fq.gz}
echo "$R1 $R2"
java -jar /usr/local/bin/trimmomatic-0.38.jar PE -threads 4 -phred33 $R1 $R2 ./02.Trimmomatic_filtering/$R1paired $R1unpaired ./02.Trimmomatic_filtering/$R2paired $R2unpaired LEADING:20 TRAILING:20 MINLEN:60
done
```

#### 03\. Align to European whitefish genome assembly

We are using [bwa](http://bio-bwa.sourceforge.net/) aligner for short
reads.

``` bash
for R1 in ./02.Trimmomatic_filtering/*.P1.fq.gz
do
R2=${R1//.P1.fq.gz/.P2.fq.gz}
base=$(basename $R1 .P1.fq.gz)
echo "$base"
bwa mem -t 4 /03.Assembly/EW_assembly.fa $R1 $R2 | samtools view -bSq 20 | \
samtools sort -o ./04.bam_alignments/$base.bam
done
```

#### 04\. Build Stacks catalog

For this project we used [STACKS
v.2.4.1](http://catchenlab.life.illinois.edu/stacks/). We are using the
`ref_map.pl` script to build a catalog with referenced aligned reads.

``` bash
ref_map.pl -T 4 --samples ./04.bam_alignments/ -o ./05.Stacks --popmap ./popmap_biogeography.txt
```

### Population genomics and phylogenetics analyses

From this point onward, we are filtering and generating datasets for
different analyses.

#### Generate a vcf file using [populations](http://catchenlab.life.illinois.edu/stacks/comp/populations.php)

Here we are telling `populations` to retain loci only if present in at
least 80% of individuals per population in at least 9 populations. We
are also removing loci with heterozygosity higher than 0.6 and minor
allele frequency below 0.05. Only one snp per locus is retained to
reduce the effect of linkage.

``` bash
populations -P ./05.Stacks -M ./popmap_biogeography2.txt -t 4 -p 9 -r 0.8 --max_obs_het 0.6 --min_maf 0.05 --write_single_snp --vcf
```

##### Filter the vcf file

Here we are using vcftools to further filter the vcf file: minimum SNP
depth of 5, minimum mean SNP depth of 10, maximum mean depth of 30, maf
of 0.05, 85% of individuals need to have the site.

``` bash
vcftools --vcf ./05.Stacks/populations.snps.vcf --minDP 5 --maxDP 30 --min-meanDP 10 --max-meanDP 30 --maf 0.05 --max-missing 0.85 --recode --recode-INFO-all --out ./5.Stacks/filtered.maf5
```

We then use the `filter_hwe_by_pop.pl` script from the
[dDocent](http://www.ddocent.com/) pipeline. This script removes sites
that are not in HWE within populations.

``` bash
./filter_hwe_by_pop.pl -v ./05.Stacks/filtered.maf5.recode.vcf -p ./popmap_biogeography2.txt -o ./05.Stacks/filtered.hwe.maf5
```

We found no loci out of HWE. Next, following an example from `dDocent`,
we look for individuals with lots of missing data and exclude from the
vcf file.

``` bash
vcftools --vcf ./05.Stacks/filtered.hwe.recode.vcf --missing-indv
mawk '$5 > 0.9' out.imiss | cut -f1 > lowDP.indv
vcftools --vcf ./05.Stacks/filtered.hwe.maf5.recode.vcf --remove lowDP.indv --recode --recode-INFO-all --out ./05.Stacks/filtered.final.maf5
```

We now generated the final vcf file, with 91 individuals and 25,751 high
quality SNPs.

#### Genetic diversity and allelic richness

##### Genetic diversity

We calculated expected and observed heterozygosity, and nucleotide
diversity using `populations`. We extracted the loci name from the final
vcf, and used them as a whitelist.

``` bash
cd ~/Desktop/biogeography/07.Population_genetics/genetic_diversity

cut -f 3 filtered.final.maf5.recode.vcf | tail -n +16 | cut -f 1 -d":" > whitelist.txt

cd ~/Desktop/biogeography/

populations -P ./05.Stacks -O ./07.Population_genetics/genetic_diversity --whitelist ./07.Population_genetics/genetic_diversity/whitelist.txt -M ./vcf_pop_file.txt -t 4 -p 8 -r 0.667 --max_obs_het 0.6 --min_maf 0.05 --smooth --vcf --bootstrap
```

##### Allelic richness

Rarefied allelic richness was calculated in the R package `hierfstat`.
We converted the final vcf to structure format using **PGDSpider**,
imported the data into R using `adegenet`.

``` r
library(adegenet); library(hierfstat)

setwd("~/Desktop/biogeography/07.Population_genetics/genetic_diversity/")

genind1 <- read.structure("populations.snps.str", n.ind = 91, n.loc = 41929, 
                          onerowperind = FALSE, col.lab = 1, 
                          NA.char = "-9", ask = FALSE, 
                          row.marknames = 1, quiet = FALSE,col.pop = 2)  
whitefish.hfstat <- genind2hierfstat(genind1, pop = genind1$pop)

AR <- allelic.richness(whitefish.hfstat, min.n = 3)

alp.rich <- mean(na.omit(AR$Ar[,1]))
bal.rich <- mean(na.omit(AR$Ar[,2]))
bwa.rich <- mean(na.omit(AR$Ar[,3]))
eck.rich <- mean(na.omit(AR$Ar[,4]))
nor.rich <- mean(na.omit(AR$Ar[,5]))
hwa.rich <- mean(na.omit(AR$Ar[,6]))
lte.rich <- mean(na.omit(AR$Ar[,7]))
lom.rich <- mean(na.omit(AR$Ar[,8]))
rta.rich <- mean(na.omit(AR$Ar[,9]))
rus.rich <- mean(na.omit(AR$Ar[,10]))
uwa.rich <- mean(na.omit(AR$Ar[,11]))
```

#### Convert the vcf file to phylip

Here we are converting the final vcf file to phylip format to be
analysed with
[RAxML](https://cme.h-its.org/exelixis/web/software/raxml/) on the
[CIPRES](https://www.phylo.org/) server. For the format conversion we
use the `vcf2phylip.py` script (Ortiz, 2019), found
[here](https://github.com/edgardomortiz/vcf2phylip).

``` bash
cd ./Desktop/biogeography/06.Phylogenetics
python vcf2phylip.py --input filtered.final.maf5.recode.vcf 
```

#### Admxiture analysis

Here we are converting the final vcf file to
[plink](zzz.bwh.harvard.edu/plink/) *bed* format to be used in
[Admixture](http://software.genetics.ucla.edu/admixture/).

``` bash
plink --vcf ./05.Stacks/filtered.final.maf5.recode.vcf --double-id --allow-extra-chr --make-bed --out ./07.Population_genetics/admixture/filtered.final

plink --bfile ./07.Population_genetics/admixture/filtered.final --double-id --allow-extra-chr --recode12 --tab --out ./07.Population_genetics/admixture/filtered.final
```

Now we can run the **admixture** analysis. First, write a loop script to
test different values of *K*. You can use any text editor. The script
looks like this:

``` bash
#!/bin/bash

for K in {2..11}
do
admixture --cv=20 $@ $K | tee log${K}.out;
mv *.P ./results;
mv *.Q ./results;
mv *.out ./results;
done
```

Now run the analysis.

``` bash
cd ./07.Population_genetics/admixture
sh ./run_admixture_loop.sh filtered.final.ped
grep -h CV ./results/log*.out

# plot results in R.
```

#### DAPC analysis in adegenet

First, we need to convert the final vcf file into plink 012 format.

``` bash
plink --vcf ./05.Stacks/filtered.final.maf5.recode.vcf --make-bed --out ./07.Population_genetics/DAPC/filtered.maf5 --allow-extra-chr 

plink --bfile ./07.Population_genetics/DAPC/iltered.maf5 --recode --tab --out ./07.Population_genetics/DAPC/filtered.maf5 --allow-extra-chr 

plink --file ./07.Population_genetics/DAPC/filtered.maf5 --recodeA --out ./07.Population_genetics/DAPC/filtered.maf5 --allow-extra-chr 

plink --file ./07.Population_genetics/DAPC/filtered.maf5 --recode12 --out ./07.Population_genetics/DAPC/filtered.maf5.12 --double-id --allow-extra-chr
```

Now we can run the
[DAPC](http://adegenet.r-forge.r-project.org/files/tutorial-dapc.pdf)
analysis in the R package adegenet.

``` r
setwd("~/Desktop/biogeography/07.Population_genetics/DAPC/")
library(adegenet)
genlight1 <- read.PLINK("filtered.maf5.raw", map.file="filtered.maf5.map")

groups <- find.clusters(genlight1, max.n.clust=11, n.pca = 90,
                        choose.n.clust = TRUE, criterion = "min")  # find most likely number of clusters

xval1 <- xvalDapc(tab(genlight1, NA.method="mean"), groups$grp, n.pca.max = 90,
                  result = "groupMean", center = TRUE, scale = FALSE, parallel = "multicore",
                  n.pca = NULL, n.rep = 100, xval.plot = TRUE)  # crossalidation to decide how many pcs to retain

dapc1 <- dapc(genlight1, pop = groups$grp, n.pca=10, n.da = 7)  # run dapc

# plot the results (Figure S2)
posterior1 <- data.frame(dapc1$posterior)
colnames(posterior1) <- c("alp","lte","lom","haw","nor","bwa","bal")
cols <- c("alp"="#874000","lte"="gold1","lom"="#2372CD","haw"="#DC8C91","nor"="#b08ea2","bwa"="#e3655b","bal"="#6b9080")

barplot(t(posterior1), col = cols, space = 0,
        xlab="Lake", ylab="Ancestry", border=NA,cex.names = 0.000001)
# the colours were then adjusted in inkscape manually!
```

#### fineRADstructure analysis

We generated a dataset in the `radpainter` format to be used by
[fineRADstructure](https://github.com/millanek/fineRADstructure) using
`populations`.

``` bash
populations -P ./05.Stacks -O ./07.Population_genetics/fineRadStructure -M ./popmap_radstructure.txt --whitelist ./05.Stacks/whitelist -t 4 -p 9 -r 0.9 --max_obs_het 0.6 --min_maf 0.05 --radpainter
```

Then we run the analysis as indicated in the manual
[here](http://cichlid.gurdon.cam.ac.uk/fineRADstructure.html).

``` bash
cd ~/Desktop/biogeography/07.Population_genetics/fineRadStructure
RADpainter paint populations.haps.radpainter
finestructure -x 300000 -y 300000 -z 3000 populations.haps_chunks.out populations.haps_chunks.mcmc.xml
finestructure -m T -x 300000 populations.haps_chunks.out populations.haps_chunks.mcmc.xml populations.haps_chunks.mcmcTree.xml
```

Results are plotted using the `fineRADstructurePlot.R` script that is
installed with the program.

#### Principal component analysis using SNPRelate

As the title indicates, a pca was run using the R Bioconductor package
`SNPRelate` (a good vignette is found
[here](http://corearray.sourceforge.net/tutorials/SNPRelate/)).

``` r
#### Principal component analysis ####

# Packages ----
library(SNPRelate);library(ggplot2);library(tidyverse);library(gridExtra);library(MASS);library(viridis)

# Set working directory ----
setwd("~/Desktop/biogeography")

# Functions ----

# Consistent plot aesthetics for PCA
theme.pca <- function() {
  theme_bw() +
    theme(panel.grid.minor = element_blank(),
          panel.grid.major = element_blank(),
          panel.background = element_rect(colour="black",fill="white", size=1),
          axis.text = element_text(size=16, color = "black"),
          axis.ticks = element_line(size = 0.5, colour = "black"),
          axis.ticks.length = unit(3, "mm"),
          axis.title.y = element_text(size = 30),
          axis.title.x = element_text(size = 30),
          axis.text.x = element_text(size=20),
          axis.text.y = element_text(size=20),
          legend.title = element_text(size = 20),
          legend.text = element_text(size = 20))
}

# Import vcf file and popdata file ----
vcf.fn <- "./7.Population_genetics/PCA/filtered.final.maf5.recode.vcf"
snpgdsVCF2GDS(vcf.fn, "./7.Population_genetics/PCA/test.final.maf5.gds", method="biallelic.only")
snpgdsSummary("./7.Population_genetics/PCA/test.final.maf5.gds")
genofile <- snpgdsOpen("./7.Population_genetics/PCA/test.final.maf5.gds")


pop_data <- read.table("vcf_pop_file.txt",header=FALSE)


# Start analysis ----
set.seed(1000)  # for reproducibility

# option to filter SNPs for LD before pca
snpset <- snpgdsLDpruning(genofile, ld.threshold=0.2,autosome.only = FALSE)  
snpset.id <- unlist(snpset)
pca <- snpgdsPCA(genofile, num.thread=2, snp.id=snpset.id, autosome.only = FALSE)

# pca with all SNPs
pca <- snpgdsPCA(genofile, num.thread=2, autosome.only = FALSE)  

# variance proportion (%)
pc.percent <- pca$varprop*100
pc.percent <- head(round(pc.percent, 2))

# Manipulate results ----
# make a data.frame
tab <- data.frame(sample.id = pca$sample.id,
                  EV1 = pca$eigenvect[,1],    # the first eigenvector
                  EV2 = pca$eigenvect[,2],
                  EV3 = pca$eigenvect[,3],
                  EV4 = pca$eigenvect[,4],
                  EV5 = pca$eigenvect[,5],
                  EV6 = pca$eigenvect[,6],
                  EV7 = pca$eigenvect[,7],# the second eigenvector
                  stringsAsFactors = FALSE)
head(tab)
# add population data
tab[,9] <- pop_data$V2
colnames(tab)[9] <- "Population"

# reorder lake names
tab$Population <- factor(tab$Population, levels = c("NOR","BAL","RUS","LOM","ECK","ALP","LTE","HAW","RTA","BWA","UWA"))

# Plot results ----

cols <- c("HAW"="#DC8C91","BWA"="#e3655b","ECK"="#74BEE9","LOM"="#2372CD","RTA"="firebrick1","LTE"="gold1","UWA"="firebrick4","ALP"="#874000","NOR"="#b08ea2","BAL"="#6b9080","RUS"="#464e47")

# PC1 and PC2
ggplot(tab, aes(x=EV1, y=EV2,col=Population)) + geom_point(size=6) + labs(x=paste("EV1",round(pc.percent, 2)[1],"%"), y = paste("EV2", round(pc.percent, 2)[2], "%")) + theme_bw() + scale_color_manual(values=cols) +
  theme(axis.title.y = element_text(size = 30),axis.title.x = element_text(size = 30),axis.text.x = element_text(size=21),axis.text.y = element_text(size=21)) +
  theme(legend.title = element_text(size=20)) + theme(legend.text = element_text(size=18))

# PC1 and PC3
ggplot(tab, aes(x=EV1, y=EV3,col=Population)) + geom_point(size=6) + labs(x=paste("EV1",round(pc.percent, 2)[1],"%"), y = paste("EV3", round(pc.percent, 2)[2], "%")) + theme_bw() + scale_color_manual(values=cols) +
  theme(axis.title.y = element_text(size = 30),axis.title.x = element_text(size = 30),axis.text.x = element_text(size=21),axis.text.y = element_text(size=21)) +
  theme(legend.title = element_text(size=20)) + theme(legend.text = element_text(size=18))



# Calculate the SNP correlations between eigenvactors and SNP genotypes:
chr <- read.gdsn(index.gdsn(genofile, "snp.chromosome"))
chr2 <- parse_number(chr)
CORR <- snpgdsPCACorr(pca, genofile, eig.which=1:4)

savepar <- par(mfrow=c(3,1), mai=c(0.3, 0.55, 0.1, 0.25))
for (i in 1:3)
{
  plot(abs(CORR$snpcorr[i,]), ylim=c(0,1), xlab="", ylab=paste("PC", i),
       col=chr2, pch="+")
}

# plot Figure S1
savepar <- par(mfrow=c(3,1), mai=c(0.3, 0.55, 0.1, 0.25))
for (i in 1:3)
{
  plot(CORR$snpcorr[i,], ylim=c(-1,1), xlab="", ylab=paste("PC", i),
       col=chr2, pch="+")
}
```

#### Genome-wide FST

We calculated genome-wide FST across populations using `vcftools`, and
then visualised the results in R.

``` bash
cd ~/Desktop/biogeography/07.Population_genetics/Fst 

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_ALP_Fst --weir-fst-pop lom.txt --weir-fst-pop alp.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_BAL_Fst --weir-fst-pop lom.txt --weir-fst-pop bal.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_NOR_Fst --weir-fst-pop lom.txt --weir-fst-pop nor.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_LTE_Fst --weir-fst-pop lom.txt --weir-fst-pop lte.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_RTA_Fst --weir-fst-pop lom.txt --weir-fst-pop rta.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_BWA_Fst --weir-fst-pop lom.txt --weir-fst-pop bwa.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_UWA_Fst --weir-fst-pop lom.txt --weir-fst-pop uwa.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out LOM_ECK_Fst --weir-fst-pop lom.txt --weir-fst-pop eck.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out ECK_ALP_Fst --weir-fst-pop eck.txt --weir-fst-pop alp.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out ECK_BAL_Fst --weir-fst-pop eck.txt --weir-fst-pop bal.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out ECK_NOR_Fst --weir-fst-pop eck.txt --weir-fst-pop nor.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out ECK_LTE_Fst --weir-fst-pop eck.txt --weir-fst-pop lte.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out ECK_RTA_Fst --weir-fst-pop eck.txt --weir-fst-pop rta.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out ECK_BWA_Fst --weir-fst-pop eck.txt --weir-fst-pop bwa.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out ECK_UWA_Fst --weir-fst-pop eck.txt --weir-fst-pop uwa.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out BAL_ALP_Fst --weir-fst-pop bal.txt --weir-fst-pop alp.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out BAL_NOR_Fst --weir-fst-pop bal.txt --weir-fst-pop nor.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out BAL_LTE_Fst --weir-fst-pop bal.txt --weir-fst-pop lte.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out BAL_RTA_Fst --weir-fst-pop bal.txt --weir-fst-pop rta.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out BAL_BWA_Fst --weir-fst-pop bal.txt --weir-fst-pop bwa.txt --fst-window-size 1000000

vcftools --vcf ../filtered.final.maf5.recode.vcf --out BAL_UWA_Fst --weir-fst-pop bal.txt --weir-fst-pop uwa.txt --fst-window-size 1000000
```

``` r
library(tidyverse)
setwd("~/Desktop/biogeography/07.Population_genetics/Fst")
lom_alp <- read.table("LOM_ALP_Fst.windowed.weir.fst", header = TRUE)
lom_bal <- read.table("LOM_BAL_Fst.windowed.weir.fst", header = TRUE)
lom_nor <- read.table("LOM_NOR_Fst.windowed.weir.fst", header = TRUE)
lom_lte <- read.table("LOM_LTE_Fst.windowed.weir.fst", header = TRUE)
lom_rta <- read.table("LOM_RTA_Fst.windowed.weir.fst", header = TRUE)
lom_uwa <- read.table("LOM_UWA_Fst.windowed.weir.fst", header = TRUE)
lom_bwa <- read.table("LOM_BWA_Fst.windowed.weir.fst", header = TRUE)
lom_eck <- read.table("LOM_ECK_Fst.windowed.weir.fst", header = TRUE)
eck_alp <- read.table("ECK_ALP_Fst.windowed.weir.fst", header = TRUE)
eck_bal <- read.table("ECK_BAL_Fst.windowed.weir.fst", header = TRUE)
eck_nor <- read.table("ECK_NOR_Fst.windowed.weir.fst", header = TRUE)
eck_lte <- read.table("ECK_LTE_Fst.windowed.weir.fst", header = TRUE)
eck_rta <- read.table("ECK_RTA_Fst.windowed.weir.fst", header = TRUE)
eck_uwa <- read.table("ECK_UWA_Fst.windowed.weir.fst", header = TRUE)
eck_bwa <- read.table("ECK_BWA_Fst.windowed.weir.fst", header = TRUE)
lom_bal <- read.table("LOM_BAL_Fst.windowed.weir.fst", header = TRUE)
eck_bal <- read.table("ECK_BAL_Fst.windowed.weir.fst", header = TRUE)
alp_bal <- read.table("BAL_ALP_Fst.windowed.weir.fst", header = TRUE)
nor_bal <- read.table("BAL_NOR_Fst.windowed.weir.fst", header = TRUE)
lte_bal <- read.table("BAL_LTE_Fst.windowed.weir.fst", header = TRUE)
rta_bal <- read.table("BAL_RTA_Fst.windowed.weir.fst", header = TRUE)
bwa_bal <- read.table("BAL_BWA_Fst.windowed.weir.fst", header = TRUE)
uwa_bal <- read.table("BAL_UWA_Fst.windowed.weir.fst", header = TRUE)

# Using Scottish populations as reference (Figure S5)
ggplot() + theme.pca() +
  stat_density(data = lom_alp, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#874000") +
  stat_density(data = lom_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#6b9080") +
  stat_density(data = lom_nor, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#b08ea2") +
  stat_density(data = lom_lte, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "gold1") +
  stat_density(data = lom_rta, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") +
  stat_density(data = lom_uwa, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") +
  stat_density(data = lom_bwa, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") +
  stat_density(data = lom_eck, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#2372CD") +
  stat_density(data = eck_alp, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#874000") +
  stat_density(data = eck_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#6b9080") +
  stat_density(data = eck_nor, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#b08ea2") +
  stat_density(data = eck_lte, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "gold1") +
  stat_density(data = eck_rta, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") +
  stat_density(data = eck_uwa, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") +
  stat_density(data = eck_bwa, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") + 
  ylim(0,3) + labs(x = "Weir & Cockerham Fst", y = "Density")

# Using Baltic population as reference (Figure S5)
ggplot() + theme.pca() +
  stat_density(data = lom_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#2372CD") +
  stat_density(data = eck_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#2372CD") +
  stat_density(data = alp_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#874000") +
  stat_density(data = nor_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "#b08ea2") +
  stat_density(data = lte_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "gold1") +
  stat_density(data = rta_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") +
  stat_density(data = uwa_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") +
  stat_density(data = bwa_bal, aes(x = WEIGHTED_FST), size = 2,geom="line", colour = "firebrick1") + 
  ylim(0,3) + labs(x = "Weir & Cockerham Fst", y = "Density")


# calculate and plot z-Fst across 1 Mb windows

# define windows to help with plotting
lom_alp$window <- 1:nrow(lom_alp)
lom_bal$window <- 1:nrow(lom_bal)
lom_nor$window <- 1:nrow(lom_nor)
lom_lte$window <- 1:nrow(lom_lte)
lom_rta$window <- 1:nrow(lom_rta)

# calculate z-fst
lom_alp$z_fst <- (lom_alp$WEIGHTED_FST - mean(lom_alp$WEIGHTED_FST)) /sd(lom_alp$WEIGHTED_FST)
lom_bal$z_fst <- (lom_bal$WEIGHTED_FST - mean(lom_bal$WEIGHTED_FST)) /sd(lom_bal$WEIGHTED_FST)
lom_nor$z_fst <- (lom_nor$WEIGHTED_FST - mean(lom_nor$WEIGHTED_FST)) /sd(lom_nor$WEIGHTED_FST)
lom_lte$z_fst <- (lom_lte$WEIGHTED_FST - mean(lom_lte$WEIGHTED_FST)) /sd(lom_lte$WEIGHTED_FST)
lom_rta$z_fst <- (lom_rta$WEIGHTED_FST - mean(lom_rta$WEIGHTED_FST)) /sd(lom_rta$WEIGHTED_FST)

# plot Figure S6
nCHR <- length(unique(lom_alp$CHROM))

lom_alp_plot <- ggplot(data = filter(lom_alp, N_VARIANTS >= 5), aes(x = window, y = z_fst, colour = as.factor(CHROM))) + 
  geom_point(size = 3) + geom_jitter(width = 0.1) + theme_bw() +
  scale_color_manual(values = rep(c("gray80", "#183059"), nCHR)) + theme(legend.position = "none") +
  theme(axis.text.x = element_text(angle = 45)) + 
  geom_hline(yintercept = 3, linetype = 'dashed') +
  ylim(0,3.5) 

lom_nor_plot <- ggplot(data = filter(lom_nor, N_VARIANTS >= 5), aes(x = window, y = z_fst, colour = as.factor(CHROM))) + 
  geom_point(size = 3) + geom_jitter(width = 0.1) + theme_bw() +
  scale_color_manual(values = rep(c("gray80", "#183059"), nCHR)) + theme(legend.position = "none") +
  theme(axis.text.x = element_text(angle = 45)) + 
  geom_hline(yintercept = 3, linetype = 'dashed') +
  ylim(0,3.5) 

lom_bal_plot <- ggplot(data = filter(lom_bal, N_VARIANTS >= 5), aes(x = window, y = z_fst, colour = as.factor(CHROM))) + 
  geom_point(size = 3) + geom_jitter(width = 0.1) + theme_bw() +
  scale_color_manual(values = rep(c("gray80", "#183059"), nCHR)) + theme(legend.position = "none") +
  theme(axis.text.x = element_text(angle = 45)) + 
  geom_hline(yintercept = 3, linetype = 'dashed') +
  ylim(0,3.5) 

lom_lte_plot <- ggplot(data = filter(lom_lte, N_VARIANTS >= 5), aes(x = window, y = z_fst, colour = as.factor(CHROM))) + 
  geom_point(size = 3) + geom_jitter(width = 0.1) + theme_bw() +
  scale_color_manual(values = rep(c("gray80", "#183059"), nCHR)) + theme(legend.position = "none") +
  theme(axis.text.x = element_text(angle = 45)) + 
  geom_hline(yintercept = 3, linetype = 'dashed') +
  ylim(0,3.5) 

lom_rta_plot <- ggplot(data = filter(lom_rta, N_VARIANTS >= 5), aes(x = window, y = z_fst, colour = as.factor(CHROM))) + 
  geom_point(size = 3) + geom_jitter(width = 0.1) + theme_bw() +
  scale_color_manual(values = rep(c("gray80", "#183059"), nCHR)) + theme(legend.position = "none") +
  theme(axis.text.x = element_text(angle = 45)) + 
  geom_hline(yintercept = 3, linetype = 'dashed') +
  ylim(0,3.5) 

library(patchwork)

lom_nor_plot / 
  lom_alp_plot /
  lom_bal_plot /
  lom_lte_plot /
  lom_rta_plot
```

#### TreeMix

We explored and visualised admixture events using TreeMix using the pop
gen dataset, testing 0-11 migration edges.

``` bash
for i in {0..11}
do
 treemix -i $@ -m $i -o ./results/edge.$i -k 100 > treemix_${i}_log &
done
```
