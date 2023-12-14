# Imputation-Workshop

This document provides basic guidelines on how to perform genotype imputation on ancient DNA datasets using a Snakemake workflow. Please, follow the instructions to download [Snakemake](https://snakemake.readthedocs.io/en/stable/getting_started/installation.html).

We will use GLIMPSE, designed to impute low-coverage whole-genome sequencing data. [Here](https://odelaneau.github.io/GLIMPSE) you can find the tool's documentation and how to install it. It is highly recommended to read [this article](https://www.nature.com/articles/s41467-023-39202-0) by Mota et al. 2022 before using GLIMPSE on ancient DNA. The tool was first published by [Rubinacci et al. 2021](https://www.nature.com/articles/s41588-020-00756-0). 


## Software requierements 
- Snakemake
- bcftools
- R (version >= 4.0)
- GLIMPSE

## Workflow overview
- Genotype likelihoods calling (bcftools 1.13)
- Genotype imputation and phasing (GLIMPSE 2.0)
- Summary statistics (per-sample)

Pre-running the pipeline step:
- Values key set-up using a configuration file in YAML format (do not modify the key name unless you change it accordingly in all Snakemake files. If you change the config filename, you will need to add this flag --configfile when running Snakemake).

The workflow is executed as follows:

```bash 
snakemake --snakefile imputation.smk -j5
```

You can find slides with basic information on how the workflow works in ```imputation_slides.pptx```
