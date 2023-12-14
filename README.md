# Inputation-Workshop

This document provides basic guidelines on how to perform genotype imputation on ancient DNA datasets using a snakemake workflow. Please, download Snakemake following the instructions here: https://snakemake.readthedocs.io/en/stable/getting_started/installation.html

We will be using GLIMPSE which has been design to impute low-coverage whole-genome sequencing data. Here you can find the documentation of the tool and how to install it: ```https://odelaneau.github.io/GLIMPSE```. It is highly recommended to read this article before using GLIMPSE on ancient DNA: ```https://www.nature.com/articles/s41467-023-39202-0```.

## Workflow overview
- Genotype likelihoods calling (bcftools 1.13)
- Genotype imputation and phasing (GLIMPSE 2.0)
- Summary statistics (per-sample)

Pre-running the pipeline step:
- Values key set-up using a configuration file in yaml format (do not modify the key name unless you change it accordingly in all snakemake files. If you change the config filename, you will need to add this flag --configfile when running snakemake).

The workflow is executed as follows:

```bash 
snakemake --snakefile imputation.smk -j5
```
