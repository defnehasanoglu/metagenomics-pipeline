# 🧬 Metagenomics Data Analysis Pipeline

This pipeline represents a baseline for metagenomics data analysis, from raw data to annotation of Metagenome-Assembled Genomes (MAGs).

---

## Overview
1. [Data Acquisition](#1-data-acquisition)
2. [Data Preprocessing](#2-data-preprocessing)
3. [Contig Assembly](#3-assembly)
4. [Coverage Statistics](#4-coverage-statistics)
5. [Creating Metagenome-Assembled Genomes (MAGs)](#5-binning-mags)
6. [Visualizing the Phylogenetic Tree](#6-tree)
7. [Annotation](#7-annotation)

---

##  1. Data Acquisition

### 1.1 SRA Run Selector
The **Sequence Read Archive (SRA)** is used to search and download metagenomics datasets. The datasets can be accessed using the accession numbers of interest at [NCBI SRA Run Selector](https://www.ncbi.nlm.nih.gov/Traces/study/).

### 1.2 Download from Terminal
Use the **SRA Toolkit** to download raw fastq files:
```bash
prefetch <accession_number>
fasterq-dump --split-files <accession_number>
```
For a list of accession numbers that can be downloaded as "SRR_Acc_List.txt" from NCBI SRA Run Selector:

```bash
prefetch --option-file SRR_Acc_List.txt
fasterq-dump --split-3 --progress --outdir fastq_files/ --option-file SRR_Acc_List.txt
gzip fastq_files/*.fastq
# --split-3 parameter to automatically split if the reads are paired-end
```

---

##  2. Data Preprocessing

### 2.1 Quality Control (QC)

Perform quality control via **FastQC** or **MultiQC** to get read quality scores, GC content, sequence duplication levels, etc.:

```bash
mkdir fastqc_out

# Running FastQC for all samples
for each in *.fastq.gz
do 
    fastqc ${each} -o fastqc_out
done
```

### 2.2 Trimming with Trimmomatic

Remove adapters and low-quality reads:

```bash
trimmomatic PE -threads 4 -phred33 \
    input_forward.fastq.gz input_reverse.fastq.gz \
    output_forward_paired.fastq.gz output_forward_unpaired.fastq.gz \
    output_reverse_paired.fastq.gz output_reverse_unpaired.fastq.gz \
    ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36
```

---

##  3. Contig Assembly 

Assemble preprocessed reads into contigs using **metaSPAdes**:

```bash
metaspades.py -1 input_forward_paired.fastq.gz -2 input_reverse_paired.fastq.gz -o metaspades_output
```

---

##  4. Coverage Statistics

Map original reads back to contigs using **BWA** and **Samtools**:

```bash
# Indexing the contigs
bwa index contigs.fasta

# Mapping
bwa mem contigs.fasta input_forward_paired.fastq input_reverse_paired.fastq > input.sam

# Reformatting and Sorting
samtools view -Sbu input.sam > input.bam
samtools sort input.bam -o input.sorted.bam
```

---

##  5. Creating Metagenome-Assembled Genomes (MAGs)

### Binning with MetaBAT2

Bin the assembled contigs into MAGs via **MetaBAT2**:

```bash
metabat2 -i metaspades_output/contigs.fasta -o metabat2_output/bin \
         -m 1500 --unbinned --saveCls --saveLog
```

### Quality Assessment (CheckM)

Perform a quality assessment after binning to estimate the completeness of MAGs via **CheckM** or **BUSCO**:

```bash
#Running CheckM
checkm lineage_wf -t 4 -x fa metabat2_output/bin checkm_output
```

---

##  6. Visualizing the Phylogenetic Tree

Visualize the results using web-based tools:

* [iTOL: Interactive Tree of Life](http://itol.embl.de/)
* [EvolView](https://www.evolgenius.info/evolview/)

> **Note:** Ensure the tree is in **Newick** format using FigTree before uploading to iTOL.

---

##  7. Annotation

Finally, annotate the MAGs to understand their functional potential via **Prokka** or **eggNOG-mapper**:

```bash
# Running Prokka
prokka --outdir prokka_output --prefix sample_name metaspades_output/contigs.fasta
```

