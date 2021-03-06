[![gemma-wrapper gem version](https://badge.fury.io/rb/bio-gemma-wrapper.svg)](https://badge.fury.io/rb/bio-gemma-wrapper)

# GEMMA with LOCO, permutations and slurm support (and caching)

![Genetic associations identified in CFW mice using GEMMA (Parker et al,
Nat. Genet., 2016)](cfw.gif)

## Introduction

GEMMA is a software toolkit for fast application of linear mixed
models (LMMs) and related models to genome-wide association studies
(GWAS) and other large-scale data sets.

This repository contains gemma-wrapper, essentially a wrapper of
GEMMA that provides support for caching the kinship or relatedness
matrix (K) and caching LM and LMM computations with the option of full
leave-one-chromosome-out genome scans (LOCO). Jobs can also be
submitted to HPC PBS, i.e., slurm.

gemma-wrapper requires a recent version of GEMMA and essentially
does a pass-through of all standard GEMMA invocation switches. On
return gemma-wrapper can return a JSON object (--json) which is
useful for web-services.

## Installation

Prerequisites are

* A recent version of [GEMMA](https://github.com/genetics-statistics/GEMMA)
* Standard [Ruby >2.0 ](https://www.ruby-lang.org/en/) which comes on
  almost all Linux systems

gemma-wrapper comes as a Ruby
[gem](https://rubygems.org/gems/bio-gemma-wrapper) and can be
installed with

    gem install bio-gemma-wrapper

Invoke the tool with

    gemma-wrapper --help

and it will render something like

```
Usage: gemma-wrapper [options] -- [gemma-options]
        --permutate n                Permutate # times by shuffling phenotypes
        --permute-phenotypes filen   Phenotypes to be shuffled in permutations
        --loco [x,y,1,2,3...]        Run full LOCO
        --input filen                JSON input variables (used for LOCO)
        --cache-dir path             Use a cache directory
        --json                       Create output file in JSON format
        --force                      Force computation
        --slurm [options]            Submit to slurm PBS
        --q, --quiet                 Run quietly
    -v, --verbose                    Run verbosely
        --debug                      Show debug messages and keep intermediate output
        --                           Anything after gets passed to GEMMA

    -h, --help                       display this help and exit
```

Alternatively, fetch a
[release](https://github.com/genetics-statistics/gemma-wrapper/releases)
of
[gemma-wrapper](https://github.com/genetics-statistics/gemma-wrapper)

Unpack it and run the tool as

    ./bin/gemma-wrapper --help

## Usage

gemma-wrapper picks up GEMMA from the PATH. To override that behaviour
use the GEMMA_COMMAND environment variable, e.g.

    env GEMMA_COMMAND=~/opt/gemma/bin/gemma ./bin/gemma-wrapper --help

to pass switches to GEMMA put them after '--' e.g.

    gemma-wrapper -v -- -h

prints the GEMMA help

## Caching of K

To compute K run the following command from the source directory (so
the data files are found):

    gemma-wrapper -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -gk \
        -debug

Run it twice to see

    /tmp/0bdd7add5e8f7d9af36b283d0341c115124273e0.log.txt CACHE HIT!

gemma-wrapper computes the unique HASH value over the command
line switches passed into GEMMA as well as the contents of the files
passed in (here the genotype and phenotype files - actually it ignores the phenotype with K because
GEMMA always computes the same K).

You can also get JSON output on STDOUT by providing the --json switch

    gemma-wrapper --json -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -gk \
        -debug

prints out something that can be parsed with a calling program

```json
{"warnings":[],"errno":0,"debug":[],"type":"K","files":[["/tmp/18ce786ab92064a7ee38a7422e7838abf91f5eb0.log.txt","/tmp/18ce786ab92064a7ee38a7422e7838abf91f5eb0.cXX.txt"]],"cache_hit":true,"gemma_command":"../gemma/bin/gemma -g test/data/input/BXD_geno.txt.gz -p test/data/input/BXD_pheno.txt -gk -debug -outdir /tmp -o 18ce786ab92064a7ee38a7422e7838abf91f5eb0"}
```

Note that GEMMA's -o (output) and --outdir switches should not be
used. gemma-wrapper stores the cached matrices in TMPDIR by
default. If you want something else provide a --cache-dir, e.g.

    gemma-wrapper --cache-dir ~/.gemma-cache -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -gk \
        -debug

will store K in ~/.gemma-cache.

### GWA

Run the LMM using the K's captured in K.json using the --input
switch

    gemma-wrapper --json --loco --input K.json -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -c test/data/input/BXD_covariates2.txt \
        -a test/data/input/BXD_snps.txt \
        -lmm 2 -maf 0.1 \
        -debug > GWA.json

Running it twice should show that GWA is not recomputed.

    /tmp/9e411810ad341de6456ce0c6efd4f973356d0bad.log.txt CACHE HIT!

### LOCO

Recent versions of GEMMA have LOCO support for a single chromosome
using the -loco switch (for supported formats check
https://github.com/genetics-statistics/GEMMA/issues/46). To loop all
chromosomes first create all K's with

    gemma-wrapper --json \
        --loco 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,X -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -a test/data/input/BXD_snps.txt \
        -gk \
        -debug > K.json

and next run the LMM's using the K's captured in K.json using the --input
switch

    gemma-wrapper --json --loco --input K.json -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -c test/data/input/BXD_covariates2.txt \
        -a test/data/input/BXD_snps.txt \
        -lmm 2 -maf 0.1 \
        -debug > GWA.json

GWA.json contains the file names of every chromosome

```json
{"warnings":[],"errno":0,"debug":[],"type":"GWA","files":[["1","/tmp/9e411810ad341de6456ce0c6efd4f973356d0bad.1.assoc.txt.log.txt","/tmp/9e411810ad341de6456ce0c6efd4f973356d0bad.1.assoc.txt.assoc.txt"],["2","/tmp/9e411810ad341de6456ce0c6efd4f973356d0bad.2.assoc.txt.log.txt","/tmp/9e411810ad341de6456ce0c6efd4f973356d0bad.2.assoc.txt.assoc.txt"]...
```

The -k switch is injected automatically. Again output switches are not
allowed (-o, -outdir)

### Permutations

Permutations can be run with and without LOCO. First create K

    gemma-wrapper --json -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -gk \
        -debug > K.json

Next, using K.json, permute the phenotypes with something like

    gemma-wrapper --json --loco --input K.json \
        --permutate 100 --permute-phenotype test/data/input/BXD_pheno.txt -- \
        -g test/data/input/BXD_geno.txt.gz \
        -p test/data/input/BXD_pheno.txt \
        -c test/data/input/BXD_covariates2.txt \
        -a test/data/input/BXD_snps.txt \
        -lmm 2 -maf 0.1 \
        -debug > GWA.json

This should get the estimated 95% (significant) and 67% (suggestive) thresholds:

    ["95 percentile (significant) ", 1.92081e-05, 4.7]
    ["67 percentile (suggestive)  ", 5.227785e-05, 4.3]

### Slurm PBS

To run gemma-wrapper on HPC use the '--slurm' switch. Example



## Copyright

Copyright (c) 2017,2018 Pjotr Prins. See [LICENSE.txt](LICENSE.txt) for further details.
