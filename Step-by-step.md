# Step-by-step explanation

Indicate the location and name of a YAML configuration file. In this case, is expected to be named 'config.yaml' 
```
configfile: 'config.yaml'
```

Bring external modules or packages
```
import pandas as pd
```

Define the number of chromosomes to impute, in this case, 22 autosomes human chromosomes. You can also add sex chromosomes to the list. 
```
CHROMS = range(1, 23)
```

Read a file specified by the path stored in the configfile under the key 'sampleIds'. The pd.read_table function is used to read the file into a Pandas DataFrame (SAMPLESFILE), and the names parameter is used to specify the column names, with 'sampleId' as the only column in this case.
```
SAMPLESFILE=pd.read_table(config['sampleIds'], names=['sampleId'])
SAMPLES=list(SAMPLESFILE['sampleId'])
```

Rule all is a special rule that defines the final output files or targets of your workflow. It specifies what should be generated when the workflow is executed. Usually, you would specify the final output of the pipeline, but there might be several targets. If the targets are up-to-date (i.e., already exist and their dependencies (input files) haven't changed), Snakemake won't execute the rules. If the targets are missing or outdated (for example, when the code is modified/updated), Snakemake will run the *necessary* rules to generate them. Several targets are separated by commas.

```
rule all:
    input:
        expand("glimpse/stats/allchr.glimpse.summary.tsv", chrom=CHROMS),
```

This is the first rule. Each rule defines how to create output files (the targets) from input files and it consists of one or several commands.

The rule compute_gl generated genotype likelihoods ('bcftools mpileup') and performs variant calling ('bcftools call'). The workflow supports multiple BAM files per sample; multiple BAM entries with the same sample ID will be automatically merged before GL calculation ('samtools merge'). You can read more on samtools and bcftools here [bcftools documentation](https://samtools.github.io/bcftools/bcftools.html). You will find out there what each argument does. 

```        
rule compute_gl:
    input:
        bam=config['bams'],
        alleles=config["panel"]["alleles"],
        ref=config["ref"],
    params:
        mq = config['mq'],
        reference = config['reference_version']
    wildcard_constraints:
        chrom="\d+",
    output:
        vcf=temp("tmpDir/{chrom}/{chrom}.{sample}.gl.vcf.gz"),
        tbi=temp("tmpDir/{chrom}/{chrom}.{sample}.gl.vcf.gz.tbi"),
        sample=temp("tmpDir/{chrom}/{sample}.sample.txt"),
# add cores and mem
    shell:
        """
        echo -e {wildcards.sample} > {output.sample}
        samtools merge - {input.bam} | bcftools mpileup -I -E -a FORMAT/DP,FORMAT/AD --ignore-RG -f {input.ref} -q {params.mq} -T {input.alleles} - | bcftools reheader -s {output.sample} | bcftools call --ploidy {params.reference} -Aim -C alleles -T {input.alleles} -Oz > {output.vcf}
        bcftools index -t {output.vcf} 
        """
```

Merge_gl merges all samples into one single file, which requires the use of the expand function, to get the input as a list of files. As you may notice, the wildcard '{sample}' won't longer exist and the allow_missing is needed to avoid snakemake complaints of missing dependencies. 

```
rule merge_gl:
    input:
        vcf=expand("tmpDir/{chrom}/{chrom}.{sample}.gl.vcf.gz", sample=SAMPLES, allow_missing=True),
        tbi=expand("tmpDir/{chrom}/{chrom}.{sample}.gl.vcf.gz.tbi",sample=SAMPLES, allow_missing=True),
    wildcard_constraints:
        chrom="\d+",
    output:
        vcf="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz",
        tbi="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz.tbi",
        lst=temp("tmpDir/{chrom}/{chrom}.gl_merge.files.txt"),
    shell:
        """
        echo {input.vcf} | perl -p -e 's/ /\\n/g' > {output.lst}
        bcftools merge -l {output.lst} -m all -Oz > {output.vcf}
        bcftools index -t {output.vcf}
        """
```
It performs imputation in chunks for computational reasons which is optional. 

```
rule chunk_glimpse:
    input:
        vcf="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz",
        tbi="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz.tbi",
    wildcard_constraints:
        chrom="\d+",
    params:
        glimpseChunk=os.path.join(config["glimpse"]["path"], 'GLIMPSE_chunk'),
        glimpseBS=config["glimpse"]["buffersize"],
        glimpseWS=config["glimpse"]["windowsize"]
    output:
        temp("tmpDir/{chrom}/{chrom}.chunks.txt"),
    log:
        "glimpse/logs/{chrom}.chunks.glimpse.log",
    shell:
        """
        {params.glimpseChunk} --input {input.vcf} --region chr{wildcards.chrom} --window-size {params.glimpseWS} --buffer-size {params.glimpseBS} --output {output} --log {log}
        """
```
```
checkpoint get_chunks_chrom:
    input:
        "tmpDir/{chrom}/{chrom}.chunks.txt",
    output:
        temp(directory("tmpDir/{chrom}/glimpse_chunks/")),
    shell:
        """
        mkdir tmpDir/{wildcards.chrom}/glimpse_chunks
        cat {input} | awk '{{print >"tmpDir/{wildcards.chrom}/glimpse_chunks/"$2"_"$1".chunk"}}'
        """
```

```
rule impute_chunk_glimpse:
    input:
        vcf="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz",
        tbi="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz.tbi",
        vcf_ref=config["panel"]["vcf"],
        genmap=config["panel"]["genmap"],
        chunk="tmpDir/{chrom}/glimpse_chunks/{chunk}.chunk",
    params:
        glimpsePhase=os.path.join(config["glimpse"]["path"], 'GLIMPSE_phase')
    wildcard_constraints:
        chrom="\d+",
    output:
        vcf=temp("tmpDir/{chrom}/{chunk}.chunk.vcf.gz"),
        tbi=temp("tmpDir/{chrom}/{chunk}.chunk.vcf.gz.tbi"),
    threads: config["glimpse"]["threads"]
    log:
        "glimpse/logs/{chrom}.{chunk}.impute_chunk.glimpse.log",
    benchmark:
        "glimpse/benchmarks/{chrom}.{chunk}.impute_chunk.glimpse.txt"
    shell:
        """
        {params.glimpsePhase} --thread {threads} --input {input.vcf} --reference {input.vcf_ref} --map {input.genmap} --input-region $(cat {input.chunk} | cut -f3) --output-region $(cat {input.chunk} | cut -f4) --output {output.vcf} --log {log} --seed $RANDOM
        bcftools index -t {output.vcf}
        """    
```

```
def aggregate_chunks(wildcards):
    checkpoint_output = checkpoints.get_chunks_chrom.get(**wildcards).output[0]

    files = expand(
        "tmpDir/{chrom}/{chunk}.chunk.{ext}",
        chrom=wildcards.chrom,
        chunk=glob_wildcards(os.path.join(checkpoint_output, "{chunk}.chunk")).chunk,
        ext=["vcf.gz", "vcf.gz.tbi"],
    )
    return files

```

```
rule ligate_chunks_glimpse:
    input:
        aggregate_chunks,
    output:
        vcf="tmpDir/{chrom}/{chrom}.ligate.vcf.gz",
        tbi="tmpDir/{chrom}/{chrom}.ligate.vcf.gz.tbi",
        lst="tmpDir/{chrom}/{chrom}.list",
    params:
        glimpseLigate=os.path.join(config["glimpse"]["path"], 'GLIMPSE_ligate')
    log:
        "glimpse/logs/{chrom}.ligate_chunks.glimpse.log",
    shell:
        """
        echo {input} | perl -p -e 's/ /\\n/g' | grep -v .tbi > {output.lst}
        {params.glimpseLigate} --input {output.lst} --output {output.vcf} --log {log}
        bcftools index -t {output.vcf}
        """
```

Phasing of genotypes

```
rule phase_glimpse:
    input:
        vcf="tmpDir/{chrom}/{chrom}.ligate.vcf.gz",
        tbi="tmpDir/{chrom}/{chrom}.ligate.vcf.gz.tbi",
    output:
        vcf=temp("tmpDir/{chrom}/{chrom}.phased.vcf.gz"),
        tbi=temp("tmpDir/{chrom}/{chrom}.phased.vcf.gz.tbi"),
    params:
        glimpsePhase=os.path.join(config["glimpse"]["path"], 'GLIMPSE_sample')
    log:
        "glimpse/logs/{chrom}.phase.glimpse.log",
    shell:
        """
        {params.glimpsePhase} --input {input.vcf} --solve --output {output.vcf} --log {log}
        bcftools index -t {output.vcf}
        """
```

This generated the final vcf file, which is both imputed and phased.

```
rule annote_vcf_phase:
    input:
        vcf_ligate="tmpDir/{chrom}/{chrom}.ligate.vcf.gz",
        tbi_ligate="tmpDir/{chrom}/{chrom}.ligate.vcf.gz.tbi",
        vcf_phase="tmpDir/{chrom}/{chrom}.phased.vcf.gz",
        tbi_phase="tmpDir/{chrom}/{chrom}.phased.vcf.gz.tbi",
    output:
        vcf_ant=temp("tmpDir/{chrom}/{chrom}.annot.vcf.gz"),
        tbi_ant=temp("tmpDir/{chrom}/{chrom}.annot.vcf.gz.tbi"),
    shell:
        """
        bcftools annotate -a {input.vcf_phase} -c FMT/GT {input.vcf_ligate} -Oz > {output.vcf_ant}
        bcftools index -t {output.vcf_ant}
        """
```

Annotation of INFO file. This can be adapted to the information you need. Read more on 'bcftools' about the possibilities. 

```
rule annote_vcf_gl:
    """ 
    Annotation of INFO field
    """
    input:
        vcf_ant="tmpDir/{chrom}/{chrom}.annot.vcf.gz",
        tbi_ant="tmpDir/{chrom}/{chrom}.annot.vcf.gz.tbi",
        vcf_gl="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz",
        tbi_gl="tmpDir/{chrom}/{chrom}.gl_merge.vcf.gz.tbi",
    output:
        vcf="glimpse/{chrom}.glimpse.vcf.gz",
        tbi="glimpse/{chrom}.glimpse.vcf.gz.tbi",
    shell:
        """
        bcftools annotate -a {input.vcf_gl} -c FMT/DP,FMT/PL -Oz {input.vcf_ant} > {output.vcf}
        bcftools index -t {output.vcf}
        """
```
Summary statistics (averaged read depth and genotype probability) per chromosome. Output is a tab-delimited file.

```
rule summarize_chrom:
    input:
        vcf="glimpse/{chrom}.glimpse.vcf.gz",
        # targetsP=config['targets']
    output:
        "glimpse/stats/chr{chrom}.glimpse.summary.tsv",
    shell:
        """
        Rscript scripts/summary.R {input.vcf} {output}
        """
```

Summary statistics per chromosome are aggregated across the genome (averaged read depth and genotype probability)

```
rule summarize_all:
    input:
        expand("glimpse/stats/chr{chrom}.glimpse.summary.tsv", chrom=CHROMS),
    output:
       "glimpse/stats/allchr.glimpse.summary.tsv",
    shell:
        """
        Rscript scripts/summaryAll.R {input} {output}
        """
```
