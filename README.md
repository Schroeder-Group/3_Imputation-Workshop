# Imputation-Workshop

## Pre-processing 

This document provides basic guidelines on how to perform genotype imputation on ancient DNA datasets using a Snakemake workflow.

We will use GLIMPSE, designed to impute low-coverage whole-genome sequencing data. Read more about [GLIMPSE](https://odelaneau.github.io/GLIMPSE) tool's documentation and how to install it. It is highly recommended to read the benchmarking article by [Mota et al. 2022](https://www.nature.com/articles/s41467-023-39202-0) before using GLIMPSE on ancient DNA. The tool was first published by [Rubinacci et al. 2021](https://www.nature.com/articles/s41588-020-00756-0). 


## Software requierements 
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
- Values key set-up using a configuration file in YAML format (do not modify the key name unless you change it accordingly in all Snakemake files. If you change the config filename, you will need to add this flag --configfile when running Snakemake).

The workflow is executed as follows:

```bash
# simple run 
snakemake --snakefile imputation.smk -j5
# if the filename is Snakefile and it located in the working directory you don't have to provide the name 
snakemake -j5
```
Specify the path using the flag ```--snakefile``` if the file is not located in your working directory. I usually have all my snakefiles in a separate directory named  ```rules```. This applied as well to the config.yaml file. The location can be indicated in the Snakefiles. 

You can find slides with basic information on how the workflow works in ```imputation_slides.pptx```

# Snakemake: text-based workflow system using python interpreter

Snakemake is very well-documented. A few useful links:
- Snakemake tutorial: https://snakemake.readthedocs.io/en/stable/tutorial/tutorial.html
- Snakemake slides: https://slides.com/johanneskoester/snakemake-tutorial 
- Snakemake advanced: https://f1000research.com/articles/10-33/v1

## Conda environments

I'd recommend to create your conda environment for each project (which will requiere different software and potentially difference versions). More information on [conda environments](https://docs.conda.io/projects/conda/en/latest/user-guide/index.html). 
```bash
# If an environmental.yaml file is given (versions specified), all dependencies and packages can be installed in a new env as follow: 
conda env create --name XXX --file environment.yaml
# When no YAML file is provided (build from scratch)
conda env create --name XXX
# activate the environment to use it
conda activate XXX
# find latest version of snakemake (or whatever other package you are interested)
conda search snakemake
# install it - one package at a time
conda install snakemake=5.15.0
# export the environment file (you will get the updated dependencies - if you had install new ones after the creation of the env)
- A. only built from user
conda env export --no-builds --from-history > environment.yaml
- B. all
conda env export > environment.yaml
```
There are some conda environments within Mjiolinr. More info in [Mjolnir documentation] https://mjolnir-ucph.readthedocs.io/en/latest/software.html#conda-environments and they are located: ```/projects/mjolnir1/apps/conda/software-version```
