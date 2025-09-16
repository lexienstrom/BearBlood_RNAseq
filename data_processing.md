# 1. Trimming reads

Trim raw reads with TrimGalore
- Fastqc of raw fastq files shows high primer contamination
- bearBlood_RNAseq_samples.tsv is a tab delimited with with the fastq name in column 1 and the path to the fastq file in column 2


Based on https://github.com/jokelley/H2Sexposure-expression/blob/master/linux_scripts.txt

```bash
# Load required modules for Trim Galore and FastQC
module load trimgalore
module load fastqc

# Change to the project directory, exit if it fails
cd /hb/groups/kelley_training/lexi/BearBlood_RNAseq || exit 1

# Create output directory for trimmed fastqs and FastQC results
mkdir -p 1_trim/trim_q0/fastqc
mkdir -p 1_trim/trim_q0_q24/fastqc

# Get the sample line corresponding to the current SLURM array task ID
LINE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" bearBlood_RNAseq_samples.tsv)

# Extract FASTQ file name and read directory from the sample line
fastq=$(echo "${LINE}" | awk '{print $1}')
readDir=$(echo "${LINE}" | awk '{print $2}')

# Run Trim Galore for adapter and quality trimming, and FastQC for QC
# --quality 0 - No quality trimming 
# --stringency 6 - Minimum overlap for adapter trimming
# --length 50 - Minimum read length to keep
trim_galore \
    --quality 0  \
    --adapter GAAGAGCGTCGTGT \
    --cores "$SLURM_CPUS_PER_TASK" \
    --fastqc_args "--noextract --nogroup --outdir 1_trim/trim_q0/fastqc" \
    --stringency 6 \
    --length 50 \
    --output_dir 1_trim/trim_q0 "$readDir"

# Trim files again with quality 24
trim_galore \
    --quality 24  \
    --adapter GAAGAGCGTCGTGT \
    --cores "$SLURM_CPUS_PER_TASK" \
    --fastqc_args "--noextract --nogroup --outdir 1_trim/trim_q0_q24/fastqc" \
    --stringency 6 \
    --length 50 \
    --output_dir 1_trim/trim_q0_q24 \
    1_trim/trim_q0/"${fastq}_trimmed.fq.gz"
```
# 2. Mapping reads to ref genome
Use STAR aligner to map to brown bear genome (GCA_023065955.2)[https://www.ncbi.nlm.nih.gov/datasets/genome/GCF_023065955.2/]
- Keep only uniquely mapped reads
- Convert to sorted BAM

```bash
# Load necessary modules
module load star

# Change to working directory
cd /hb/groups/kelley_training/lexi/BearBlood_RNAseq || exit 1

# Create output directory for STAR mapped files
mkdir -p 2_STAR/mapped

# Read the sample information from the TSV file based on SLURM_ARRAY_TASK_ID
# Each line in bearBlood_RNAseq_samples.tsv should have two columns: sample_name and full_path_to_fastq
LINE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" bearBlood_RNAseq_samples.tsv)
fastq=$(echo "${LINE}" | awk '{print $1}')

# Run STAR
# max read length is ~331, so sjdbOverhang should be set to 330 (max read length - 1)
# indexed genome using:
# STAR --runMode genomeGenerate \
#    --runThreadN 12 \
#    --genomeDir indexed_genome \
#    --genomeFastaFiles "${fastaFile}" \
#    --sjdbGTFfile "${gtfFile}" \
#    --sjdbOverhang 330

STAR \
    --runThreadN 4 \
    --genomeDir 2_STAR/indexed_genome \
    --sjdbGTFfile /hb/home/aenstrom/ursus_arctos_genome/GCF_023065955.2_UrsArc2.0_genomic.gtf \
    --sjdbOverhang 330 \
    --outFilterMultimapNmax 1 \
    --twopassMode Basic \
    --readFilesIn 1_trim/trim_31/"${fastq}_trimmed_trimmed.fq.gz" \
    --readFilesCommand zcat \
    --outFileNamePrefix 2_STAR/mapped/"${fastq}_" \
    --outSAMtype BAM SortedByCoordinate
```

# 3. Quantify gene-level read counts
Use featureCounts from Subread
```bash
# Change to working directory
cd /hb/groups/kelley_training/lexi/BearBlood_RNAseq || exit 1

# Create output directory for STAR mapped files
mkdir -p 3_featureCounts

# Run featureCounts
# Quanitfy gene-level read counts
featureCounts -F 'GTF' -T 4 -t exon -g gene_id \
    -G /hb/home/aenstrom/ursus_arctos_genome/GCF_023065955.2_UrsArc2.0_genomic.fna \
    -a /hb/home/aenstrom/ursus_arctos_genome/GCF_023065955.2_UrsArc2.0_genomic.gtf \
    -o 3_featureCounts/brownBear_Blood_RNAseq_featureCounts_output.txt \
    2_STAR/mapped/*_Aligned.sortedByCoord.out.bam
```
Etc