# Imputation-Workshop

#NB add note about conda

This document provides basic guidelines on how to perform genotype imputation on ancient DNA datasets using a Snakemake workflow. Please, follow the instructions to download [Snakemake](https://snakemake.readthedocs.io/en/stable/getting_started/installation.html).

We will use GLIMPSE, designed to impute low-coverage whole-genome sequencing data. Read more about [GLIMPSE](https://odelaneau.github.io/GLIMPSE) tool's documentation and how to install it. It is highly recommended to read the benchmarking article by [Mota et al. 2022](https://www.nature.com/articles/s41467-023-39202-0) before using GLIMPSE on ancient DNA. The tool was first published by [Rubinacci et al. 2021](https://www.nature.com/articles/s41588-020-00756-0). 


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

# Snakemake: text-based workflow system using python interpreter

Snakemake is very well-documented. A few useful links:
- Snakemake tutorial: https://snakemake.readthedocs.io/en/stable/tutorial/tutorial.html
- Snakemake slides: https://slides.com/johanneskoester/snakemake-tutorial 
- Snakemake advanced: https://f1000research.com/articles/10-33/v1

