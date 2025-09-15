# Trimming reads

Trim raw reads with TrimGalore
- Fastqc of raw fastq files shows high primer contamination
- bearBlood_RNAseq_samples.tsv is a tab delimited with with the fastq name in column 1 and the path to the fastq file in column 2

## 1. Trimming method 1 - setting quality to zero to run trimgalore

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
# --stringency 5 - Minimum overlap for adapter trimming
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
