# Targeted ctDNA Variant Analysis in Metastatic Colorectal Cancer

## Project overview
This repository contains a proposed bioinformatics workflow for analyzing circulating tumor DNA (ctDNA) sequencing data from a single metastatic colorectal cancer patient, based on the study:
[Circulating tumor DNA sequencing in colorectal cancer patients treated with first-line chemotherapy with anti-EGFR](https://www.nature.com/articles/s41598-021-95345-4)

Raw sequencing data were obtained from **NCBI BioProject PRJNA714799**.

> FASTQ files are downloaded locally and are not included in this repository.

---

## Proposed Data Analysis Pipeline Overview

- Data acquisition, exploratory data analysis, and sample selection
- FASTQ quality control (FastQC + MultiQC)
- Alignment to GRCh38 (BWA-MEM)
- ctDNA-optimized variant calling (VarScan2)
- Functional and clinical annotation (SnpEff, SIFT, PolyPhen, ClinVar, COSMIC)
- Longitudinal variant comparison and visualization of allele frequency dynamics

---

## Environment Setup Before Analysis

Set up a reproducible computational environment with all required tools.

### Tools
- Conda
- FastQC
- MultiQC
- BWA
- SAMtools
- VarScan2
- Python (pandas, matplotlib, seaborn)
- Jupyter Notebook

### Command
```bash
conda create -n ctDNA_analysis python=3.10
conda activate ctDNA_analysis
conda install -c bioconda fastqc multiqc bwa samtools varscan
conda install -c conda-forge pandas matplotlib seaborn jupyter
```

---

## 1. Data Acquisition & Exploratory Data Analysis

Identify appropriate samples and download raw sequencing data.

### Tools
- Entrez Direct
- SRA Toolkit
- Python (pandas)

### Commands

```bash
# Download RunInfo metadata
esearch -db sra -query PRJNA714799 | efetch -format runinfo > PRJNA714799_runinfo.csv
```

```python
# Download and explore metadata
import pandas as pd
df = pd.read_csv("PRJNA714799_runinfo.csv")
df.head()
```

⚠️ **Important inconsistency**  
Supplementary Table 1 in the paper defines samples as CTDC01, CTDC01-2, CTDC01-5, etc., corresponding to baseline, response, and progression. These sample IDs do not appear directly in the SRA RunInfo table, requiring additional exploration of metadata fields or author clarification.

```python
# Create new column with names unique for each patient
df['SampleBase'] = df['SampleName'].str.split('-').str[0]
print(df[['SampleName', 'SampleBase']].head(8))

# Select one patient with multiple timepoints and display SRR IDs
df_patient = df[df['SampleBase'] == 'CTC325']
df_patient[['Run', 'SampleName', 'SampleBase']]
```

Example SRR IDs for the selected patient:
- SRR13974041
- SRR13974040
- SRR13974039

```bash
# Download FASTQ files
prefetch SRR13974041
fasterq-dump SRR13974041 --split-files --threads 8 --gzip
```

---

## 2. Quality Control (QC)

Assess sequencing quality and summarize QC metrics.

### Tools
- FastQC
- MultiQC

### Commands
```bash
fastqc *.fastq.gz -o qc/
multiqc qc/ -o qc_summary/
```

### Output
- Individual FastQC reports (.html)
- [Combined MultiQC report](https://github.com/ekhlebus/ctDNA_KRAS_variant_analysis/blob/main/results/multiqc_report.html)

---

## 3. Alignment to Reference Genome

Align reads to the human reference genome GRCh38, prepare analysis-ready BAM files, and collect summary statistics.

### Tools
- BWA-MEM
- SAMtools

### Commands
```bash
bwa mem -t 8 GRCh38.fa sample_R1.fastq.gz sample_R2.fastq.gz | samtools sort -o sample.sorted.bam
samtools index sample.sorted.bam
samtools flagstat sample.sorted.bam
```

---

## 4. Variant Calling (ctDNA-Optimized)

Call somatic variants with high sensitivity to low allele fractions.

### Tools
- VarScan2 ((well suited for this task because it is explicitly designed to detect low-frequency variants, common in ctDNA))
- SAMtools

```bash
# Generate mpileup
samtools mpileup -f GRCh38.fa sample.sorted.bam > sample.mpileup

# Call variants
java -jar VarScan.v2.4.4.jar mpileup2cns sample.mpileup   --min-coverage 100   --min-var-freq 0.005   --p-value 0.05   --output-vcf 1 > sample.varscan.vcf
```

**Notes:**  
Parameters are tuned to retain variants down to ~0.5% variant allele frequency (VAF), appropriate for ctDNA analysis.

---

## 5. Variant Annotation

Annotate variants for functional and clinical relevance.

### Tools
- SnpEff, SIFT, PolyPhen, ClinVar, COSMIC
- Python (custom filtering)

### Annotation Focus
- Protein change
- Known clinical relevance
- Association with anti-EGFR resistance

### Output
- Annotated variant table (CSV / DataFrame)

---

## 6. Comparative Analysis, Visualization & Interpretation

Compare variant profiles across timepoints.

### Tools
- Python (pandas) for analysis
- Python (matplotlib, seaborn) for visualization

### Comparisons
- Presence / absence
- Allele frequency shifts between conditions
- Allele frequency over time plots
- Pre- vs post-treatment comparison charts

---

## Potential Key Findings (Summary)

- Identification of variants associated with anti-EGFR resistance
- Evidence of variant emergence and allele frequency shifts during treatment
- Consistency with findings reported in the original study
