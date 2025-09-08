# Trimming reads

Trim raw reads with TrimGalore
- Fastqc of raw fastq files shows high primer contamination
- bearBlood_RNAseq_samples.tsv is a tab delimited with with the fastq name in column 1 and the path to the fastq file in column 2

1. Trimming method 1 - setting quality to zero to run trimgalore
Based on https://github.com/jokelley/H2Sexposure-expression/blob/master/linux_scripts.txt

```bash
module load trimgalore
module load fastqc

cd /hb/groups/kelley_training/lexi/BearBlood_RNAseq || exit 1

mkdir -p 1_trim/fastqc
mkdir -p 1_trim/qual_0

LINE=$(sed -n "${SLURM_ARRAY_TASK_ID}p" bearBlood_RNAseq_samples.tsv)

fastq_file=$(echo "${LINE}" | awk '{print $1}')
readDir=$(echo "${LINE}" | awk '{print $2}')

trim_galore \
    --quality 0  \
    --cores "$SLURM_CPUS_PER_TASK" \
    --fastqc_args "--noextract --nogroup --outdir 1_trim/fastqc" \
    --stringency 5 \
    --length 50 \
    --output_dir 1_trim/qual_0 "$readDir"
```
