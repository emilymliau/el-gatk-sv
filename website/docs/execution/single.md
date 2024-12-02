---
title: Single-sample
description: Run the pipeline on a single sample
sidebar_position: 3
slug: single
---

## Introduction

**Extending SV detection to small datasets**

The Single Sample pipeline is designed to facilitate running the methods developed for the cohort-mode GATK-SV pipeline on small data sets or in
clinical contexts where batching large numbers of samples is not an option. To do so, it uses precomputed data, SV calls,
and model parameters computed by the cohort pipeline on a reference panel composed of similar samples. The pipeline integrates this
precomputed information with signals extracted from the input CRAM file to produce a call set similar in quality to results
computed by the cohort pipeline in a computationally tractable and reproducible manner.


## Terra workspace
Users should clone the [Terra single-sample workspace](https://app.terra.bio/#workspaces/help-gatk/GATK-Structural-Variants-Single-Sample)
which is configured with a demo sample to run on. 
Refer to the following sections for instructions on how to run the pipeline on your data using this workspace.

## Data

### Case sample

The default workspace includes a NA12878 input WGS CRAM file configured in the workspace samples data table. This file is part of the
high coverage (30X) [WGS data](https://www.internationalgenome.org/data-portal/data-collection/30x-grch38) for the 1000
Genomes Project samples generated by the New York Genome Center and hosted in
[AnVIL](https://app.terra.bio/#workspaces/anvil-datastorage/1000G-high-coverage-2019).

### Reference panel

The reference panel configured in the workspace consists of data and calls computed from 156 publicly available samples
chosen from the NYGC/AnVIL 1000 Genomes high coverage data linked above.

Inputs to the pipeline for the reference panel include:
- A precomputed SV callset VCF, and joint-called depth-based CNV call files
- Raw calls for the reference panel samples from Manta and WHAM
- Trained models for calling copy number variation in GATK gCNV case mode
- Parameters learned by the cohort mode pipeline in training machine learning models on the reference panel samples.

These resources are primarily configured in the "Workspace Data" for this workspace. However, several of the resources need
to be passed  to the workflow as large lists of files or strings. Due to Terra limitations on uploading data containing lists to the
workspace data table, these resources are specified directly in the workflow configuration.

### Reference resources

The pipeline uses a number of resource and data files computed for the hg38 reference:
- Reference sequences and indices
- Genome annotation tracks such as segmental duplication and RepeatMasker tracks
- Data used for annotation of called variants, including GenCode gene annotations and gnomAD site allele frequencies.

## Single sample workflow

### What does it do?

The workflow `gatk-sv-single-sample` calls structural variations on a single input CRAM by running the GATK-SV Single Sample Pipeline end-to-end.

#### What does it require as input?

The workflow accepts a single CRAM or BAM file as input, configured in the following parameters:

|Input Type|Input Name|Description|
|---------|--------|--------------|
|`String`|`sample_id`|Case sample identifier|
|`File`|`bam_or_cram_file`|Path to the GCS location of the input CRAM or BAM file|
|`String`|`batch`|Arbitrary name to be assigned to the run|

#### Additional workspace-level inputs

- Reference resources for hg38
- Input data for the reference panel
- The set of docker images used in the pipeline.

Please contact GATK-SV developers if you are interested in customizing these
inputs beyond their defaults.

#### What does it return as output?

|Output Type|Output Name|Description|
|---------|--------|--------------|
|`File`|`final_vcf`|SV VCF output for the pipeline. Includes all sites genotyped as variant in the case sample and genotypes for the reference panel. Sites are annotated with overlap of functional genome elements and allele frequencies of matching variants in gnomAD|
|`File`|`final_vcf_idx`|Index file for `final_vcf`|
|`File`|`final_bed`|Final output in BED format. Filter status, list of variant samples, and all VCF INFO fields are reported as additional columns.|
|`File`|`metrics_file`|Metrics computed from the input data and intermediate and final VCFs. Includes metrics on the SV evidence, and on the number of variants called, broken down by type and size range.|
|`File`|`qc_file`|Quality-control check file. This extracts several key metrics from the `metrics_file` and compares them to pre-specified threshold values. If any QC checks evaluate to FAIL, further diagnostics may be required.|
|`File`|`ploidy_matrix`|Matrix of contig ploidy estimates computed by GATK gCNV.|
|`File`|`ploidy_plots`|Plots of contig ploidy generated from `ploidy_matrix`|
|`File`|`non_genotyped_unique_depth_calls`|This VCF file contains any depth based calls made in the case sample that did not pass genotyping checks and do not match a depth-based call from the reference panel. If very high sensitivity is desired, examine this file for additional large CNV calls.|
|`File`|`non_genotyped_unique_depth_calls_idx`|Index file for `non_genotyped_unique_depth_calls`|
|`File`|`pre_cleanup_vcf`|VCF output in a representation used internally in the pipeline. This file is less compliant with the VCF spec and is intended for debugging purposes.|
|`File`|`pre_cleanup_vcf_idx`|Index file for `pre_cleanup_vcf`|

#### Example time and cost run on sample data

|Sample Name|Sample Size|Time|Cost $|
|-----------|-----------|----|------|
|NA12878|18.17 GiB|23hrs|~$7.34|

#### To use this workflow on your own data

If you would like to run this workflow on your own samples (which must be medium-to-high coverage WGS data):

- Clone the [workspace](https://app.terra.bio/#workspaces/help-gatk/GATK-Structural-Variants-Single-Sample) into a Terra project you have access to.
  Select `us-central1` for the region. If you must use a different region, you will need to copy all GATK-SV docker images to the other region
  before running the pipeline. See the [docker images section](/docs/gs/dockers#regions-important) for details.
- In the cloned workspace, upload rows to the Sample and (optionally) the Participant Data Table that describe your samples.
  Ensure that the rows you add to the Sample table contain the columns `sample_id` and `bam_or_cram_file` are populated appropriately.
- There is no need to modify values in the workspace data or method configuration. If you are interested in modifying the reference
  genome resources or reference panel, please contact the GATK team for support as listed below.
- Launch the workflow from the "Workflows" tab, selecting your samples as the inputs.

#### Quality control

Please check the `qc_file` output to screen for data quality issues. This file provides a table of quality metrics and 
suggested acceptable ranges. 

:::warning
If a metric exceeds the recommended range, all variants will be automatically flagged with 
a non-passing `FILTER` status in the output VCF.
:::