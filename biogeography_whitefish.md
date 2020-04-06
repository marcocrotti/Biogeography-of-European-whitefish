Biogeography of European whitefish
================
Marco Crotti
06 April, 2020

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
ref_map.pl -T 4 --samples ./04.bam_alignments/ -o ./05.Stacks --popmap ./popmap_2_biogeography.txt
```

### Population genomics and phylogenetics analyses

From this point onward, we are filtering and generating datasets for
different analyses.
