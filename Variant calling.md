## SNP calling from Ornatus dragon genome
### Dataset
There were 26 paired end reads downloaded from AGRF (non-public data), Tash's project. The raw reads are backed up in pshell
### QC and trimming
```
fastqc *.fastq.gz -o out_dir 
fastp -i input1.fastq.gz -o output1.fastq.gz -I input2.fastq.gz -O output2.fastq.gz
```
### Alignment
Align the trimmed reads to the reference and then sort the files. I used bowtie2 and samtools to do it. Build a index for the reference first, align and sort using samtools. 
```
bowtie2-build reference.fa output_name 
bowtie2 --end-to-end --sensitive -x /scratch/pawsey0149/supadhyaya/ornate_dragon/ornate_dragon {-1 ${FILENAME1} -2 ${FILENAME2}} | samtools sort -o ${FILENAME%%_R1.trimmed.fastq.gz}_sorted.bam
```
### Variant Calling
I performed variant calling using bcftools on snakemake.
### Indexing
The reference file needs to be indexed before running mpileup. For indexing I used snakemake samtools wrapper.
```
#check if logfile exists or make new if it doesn't
import os.path

if not os.path.exists("slurm_logs"):
    os.mkdir("slurm_logs")

#define samples used for the whole process
IDS, = glob_wildcards("{id}.fa")

#this rule collects results and is the final step of the code
#once the input of rule all is reached, snakemake knows that it's finished
rule all:
        input:
                expand("{id}.fa.fai", id=IDS)

                                                                                                                                                                                                        
rule samtools_index:
    input:
        "{id}.fa",
    output:
        "{id}.fa.fai",
    log:
        "{id}.log",
    params:
        extra="",  # optional params string
    wrapper:
        "v1.30.0/bio/samtools/faidx" 
```
#### Calling
The snakmake wrapper takes aligned bam files as input and .fa file as reference, uses bcftools mpileup and call function to call SNPs on the files and returns .bcf files back as output. 

```
# check if logfile exists or make new if it doesn't
import os.path

if not os.path.exists("slurm_logs"):
    os.mkdir("slurm_logs")

#define samples used for the whole process
IDS, = glob_wildcards("mapped/{id}.bam")

#this rule collects results and is the final step of the code
#once the input of rule all is reached, snakemake knows that it's finished
rule all:
        input:
                expand("vcf/{id}.view.vcf", id=IDS)

#first step for snp calling using bcf tools
rule bcftools_mpileup:
    input:
        alignments="mapped/{id}.bam",
        ref="reference/reference.fa",  # this can be left out if --no-reference is in options
        index="reference/reference.fa.fai",
    output:
        pileup="pileups/{id}.pileup.bcf",
    params:
        extra="--max-depth 100 --min-BQ 15",
    log:
        "logs/bcftools_mpileup/{id}.log",
    resources:time='15:00:00',
    benchmark:
        "benchmarks/bcftools_mpileup/{id}.benchmark",
    wrapper:
        "v1.30.0/bio/bcftools/mpileup"

rule bcftools_call:
    input:
        pileup="pileups/{id}.pileup.bcf",
    output:
        calls="calls/{id}.calls.bcf",
    params:
        caller="-m",
        extra="--ploidy 2 --prior 0.001"
    log:
        "logs/bcftools_call/{id}.log",
    resources:time='15:00:00',
    wrapper:
        "v1.30.0/bio/bcftools/call"

rule bcf_view_o_vcf:
    input:
        "calls/{id}.calls.bcf",
    output:
        "vcf/{id}.view.vcf",
    log:
        "logs/{id}.view.vcf.log",
    params:
        extra="",
    resources:time='10:00:00',
    wrapper:
        "v1.30.0/bio/bcftools/view"
```

You get 26 calls.bcf files which have to be merged. Each of these files need to be indexed, merged and converted to .vcf using bcftools. 
```
bcftools index *.bcf
bcftools merge *calls.bcf -o ../merge/ornatedragon_merged.vcf -O v
```
For separate VCFs for each sample you convert them separately, and can check stats for each
```
bcftools view ${FILENAME} -O v -o ${FILENAME%%.calls.bcf}.vcf
bcftools stats filename > output.txt
```

### Filtering
``` vcftools --vcf ${FILENAME} --max-alleles 2 --max-missing 0.1 --minQ 30 --recode --out ${FILENAME%%.vcf} ```



