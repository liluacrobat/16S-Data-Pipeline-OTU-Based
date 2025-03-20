# 16S rRNA Data Processing Pipeline

## Overview
This pipeline provides instructions for processing 16S rRNA sequence data using Mothur on the CCR cluster. The pipeline includes data transfer, sequence quality assessment, trimming, and OTU-based analysis.
This pipeline could be used for mice microbiomes and the samples whose microbiomes are not well characterized.

## 1. Transfer Data from Box to CCR
Use Globus to transfer sequences from Box to the destination directory in CCR. The sequence files should be in `fastq.gz` format and placed in a folder named `fastq`.

Detailed instructions for data transfer: [CCR Documentation](https://docs.ccr.buffalo.edu/en/latest/hpc/data-transfer/)

**Helpful Information:**
- **Group permanent directory:** `/projects/academic/pidiazmo`
- **Group temporary directory (files deleted after 60 days):** `/vscratch/grp-pidiazmo`

## 2. Submitting Jobs in CCR
CCR imposes CPU time limits for terminal commands. Most processing should be submitted as batch jobs.

### Example Shell Script for Job Submission
```bash
#!/bin/bash -l
#SBATCH --partition=general-compute
#SBATCH --qos=general-compute
#SBATCH --cluster=ub-hpc
#SBATCH --time=71:00:00
#SBATCH --nodes=1
#SBATCH --cpus-per-task=12
#SBATCH --mem=120G
#SBATCH --job-name="Mothur"
#SBATCH --mail-user=lli59@buffalo.edu
#SBATCH --output=Mothur-pipeline.log
#SBATCH --mail-type=ALL
```
**Variable Explanations:**
- `time`: Maximum job duration (up to 72 hours).
- `nodes`: Number of compute nodes used.
- `cpus-per-task`: Number of CPUs requested (should be greater than the number of threads running in parallel).
- `mem`: Requested memory.
- `job-name`: Identifier for the job.
- `mail-user`: Email for job notifications.

### Job Management Commands
- Submit a job: `sbatch job_file.sh`
- Check job status: `squeue -u UB_IT_NAME`

More details: [CCR Job Submission](https://docs.ccr.buffalo.edu/en/latest/hpc/jobs/)

## 3. Decompress Sequence Files and Generate Quality Reports
### Decompress Files
```bash
for x in $(ls *.gz); do gunzip $x; done
```

### Generate Sequence Quality Reports
```bash
module load fastqc\module load gcc/11.2.0 openmpi/4.1.1
module load multiqc/1.14
mkdir fastqc
fastqc -o fastqc *.fastq
cd fastqc
multiqc . --interactive
cd ..
```
**Output:** `fastqc/multiqc_report.html`

**Key Quality Report Sections:**
- **Sequence Counts**
- **Sequence Quality Histograms**
- **Per Base N Content** (If sequences contain many `N` bases at the beginning, trimming is required.)

### Trim Sequences (if necessary)
```bash
mkdir trimmed
for x in $(ls *_R1_001.fastq); do /projects/academic/pidiazmo/projectsoftwares/fastx/bin/fastx_trimmer -f 7 -i $x -o trimmed/$x; done
for x in $(ls *_R2_001.fastq); do /projects/academic/pidiazmo/projectsoftwares/fastx/bin/fastx_trimmer -f 7 -i $x -o trimmed/$x; done
```
**Parameter:** `-f`: Number of bases trimmed from the beginning.

## 4. Create Processing Directory and Link Sequence Files
```bash
mkdir Mothur_processing
cd Mothur_processing
ln -s ../fastq/*.fastq .
```

## 5. Generate File List for Mothur Processing
```bash
MOTHURPATH='/projects/academic/pidiazmo/projectsoftwares/mothur/v1.48.1'
$MOTHURPATH/mothur "#make.file(inputdir=., type=fastq, prefix=plate_16S)"
```
Open `plate_16S.files` in Excel to verify correct sample names.

## 6. Submit Mothur_OTU_16S_S1.sh Job
```bash
sbatch Mothur_OTU_16S_S1.sh
```
After completion, check `align_summary.txt` to determine start and end parameters for `screen.seqs` in `Mothur_OTU_16S_S2.sh`. Adjust `REF_FA` and `REF_TAX` based on project data type.

## 7. Submit Mothur_OTU_16S_S2.sh Job
```bash
sbatch Mothur_OTU_16S_S2.sh
```

---

This pipeline ensures standardized and efficient processing of 16S rRNA sequencing data using CCR resources.

