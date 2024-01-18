# Imputation-Workshop

This document provides basic guidelines for performing genotype imputation on ancient DNA datasets using a Snakemake workflow. Imputation refers to the process of predicting unobserved genetic variants in the genome of, in this case, ancient samples based on the known variants in modern (and very large) reference panels. If you are not familiar with Snakemake, there is guidance on how to get started and run the pipelines in this README.

Please read [this guide](Step-by-step.md) to read what each step (rule) in the pipeline does (corresponding to the file ```Step-by-step.md```). 


## Pre-processing 

Note that this pipeline assumes you already have mapped your data to a reference. Important: use the same version of the reference genome for imputation. 

## Software requierements 

We will use GLIMPSE which was originally designed to impute low-coverage whole-genome sequencing data. The tool was first published by [Rubinacci et al. 2021](https://www.nature.com/articles/s41588-020-00756-0). It is highly recommended to read the benchmarking article by [Mota et al. 2022](https://www.nature.com/articles/s41467-023-39202-0) before using GLIMPSE on ancient DNA. Read more about [GLIMPSE](https://odelaneau.github.io/GLIMPSE) tool's documentation and how to install it.

Please, follow the instructions to download [Snakemake](https://snakemake.readthedocs.io/en/stable/getting_started/installation.html). 

- Snakemake 
- bcftools
- R (version >= 4.0)
- GLIMPSE

## Workflow overview
- Genotype likelihoods calling (bcftools 1.13)
- Genotype imputation and phasing (GLIMPSE 2.0)
- Summary statistics (per-sample)

Pre-running the pipeline step:
- Values key set-up using a configuration file in YAML format (do not modify the key names unless you change it accordingly in all Snakemake files)

The workflow is executed as follows:

```bash
# simple run 
snakemake --snakefile rules/imputation.smk -j5
# If the filename is named 'Snakefile' and is located in the working directory you don't have to provide the name 
snakemake -j5
```
Specify the path using the --snakefile flag if the file is not located in your working directory. This applies as well to the config.yaml file using the ```--configfile``` flag. Its location can also be indicated in the Snakefiles. Tip: I usually have all my Snakefiles in a separate directory named rules, and the config.yaml in the folder where I execute the Snakemake command. 

## Output files
- Merged VCF files with imputed and phased genotypes
- Summary statistics containing genotype probabilities (GP) and read depth for further filtering
- Optional: conversion of merged VCF files to PLINK format

The VCF file contains a variant-level ```INFO``` score (imputation info quality score) which is calculated from the samples' GP (genotype probabilities) values and is specific to the sample set included in the VCF. If samples are removed afterwards (for example, due to quality control reasoning such as being too low coverage or another sample's relative), it is recommended to recalculate the INFO value for the new set (the value will change slightly). Further filtering of SNPs for MAF and/or INFO score is advisable (e.g. INFO >= 0.5) for population genetics studies.

You can find slides with basic information on how the workflow works in ```imputation_slides.pptx```

## Snakemake: text-based workflow system using Python interpreter

Before running a snakemake pipeline, I would *always* recommend running a dry-run beforehand. This step will simulate the execution of a workflow without actually running the specified rules or generating any output files. It is very useful for:
- debugging and validation: errors, missing files etc. will be printed
- preview the workflow steps: ensuring the sequence of steps is correct
  
```bash
# dry-run. The -np flags specify that job execution will be simulated (-n) and the individual rule commands printed (-p)
snakemake -np 
```
Snakemake is very well-documented. A few useful links:
- Snakemake tutorial: https://snakemake.readthedocs.io/en/stable/tutorial/tutorial.html
- Snakemake slides: https://slides.com/johanneskoester/snakemake-tutorial 
- Snakemake advanced: https://f1000research.com/articles/10-33/v1

## Conda environments

I recommend creating your conda environment for each project (which will require different software and potentially different versions - isolation) and will make your project reproducible and easy to share with others and collaborators. More information on [conda environments](https://docs.conda.io/projects/conda/en/latest/user-guide/index.html). 

Some useful commands to get started:

```bash
# List all the current channels
conda config --show channels
# Add new channels to the front of the priority list (you only need to do this once)
conda config --prepend channels bioconda
conda config --prepend channels conda-forge
# List the available environments
conda env list
# If an environmental.yaml file is given (versions specified), all dependencies and packages can be installed in a new env as follows: 
conda env create --name XXX --file environment.yaml
# create an empty new environment
conda env create --name XXX
# Activate the environment to use it
conda activate XXX
# find the latest version of Snakemake (or whatever other package you are interested in)
conda search snakemake
# install it - one package at a time
conda install snakemake=5.15.0
# Export the environment file (you will get the updated dependencies - if you had installed new ones after the creation of the env)
- A. only built from user
conda env export --no-builds --from-history > environment.yaml
- B. all
conda env export > environment.yaml
# Deactivate an environment
conda deactivate
# Delete environment
conda env remove --name XXX
```
There are some conda environments within Mjiolinr. More info in [Mjolnir documentation] https://mjolnir-ucph.readthedocs.io/en/latest/software.html#conda-environments and they are located: ```/projects/mjolnir1/apps/conda/software-version```
