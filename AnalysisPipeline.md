# Analysis Pipeline for Variant Discovery and GWAS of *Paramormyrops kingsleyae* EOD Structure

This file is intended as a map of the pipeline; it should grow with the pipeline as I build it to act as aide-memoire for me and JG.

## Overview

The roughest outline goes like this:

**QC** --> **Alignment** --> **Sorting/Indexing** --> **Variant Calling** --> **Statistical Analysis**

One added level of complexity is to be found in the **Variant Calling** stage given the need to bootstrap ourselves a SNP database in order for GATK to work optimally.

Once data reaches the **Statistical Analysis** stage it may become necessary for the pipeline to branch as it seems likely that I'll need to use more than one program to generate the statistics that we're aiming at...

## Map

   |  Stage   |   Script    |   Tool(s)   |   Options   |   Input   |   Output    |   Description
---|----------|-------------|-------------|-------------|-----------|-------------|---------------
1. | **QC** | `trimmomatic_array.qsub ` | Trimmomatic/0.32 | `ILLUMINACLIP:Illumina_adapters.fa:2:30:10 HEADCROP:10 MAXINFO:50:0.5` | `Sample_library_Lane_RX_Ye.fastq.gz` | `Sample_library_Lane_RX_Ye.trimmed.fq` | gunzips ..fastq.gz files and trims
2. | **Alignment** | `alignment_array.qsub` | bwa/0.7.12.r1044 | `bwa mem -M -R ..` | `Sample_library_Lane_RX_Ye.trimmed.fq` & `Reference.fa` | `Sample_library_Lane_RX_Ye.aligned.sam` | pulls details from filenames to make an `@RG` tag and then runs alignment
3. | **Sorting/Indexing** | `deduplication_array.qsub` | picardTools/1.89 | `SORT_ORDER=coordinate` | `Sample_library_Lane_RX_Ye.aligned.sam` | `Sample_library_Lane_RX_Ye.dedup.bam` | SortSam.jar sorts, MarkDuplicates.jar marks duplicates & BuildBamIndex.jar indexes
4. | | `indel_realign_array.qsub` | GATK/3.4.46 | | `Sample_library_Lane_RX_Ye.dedup.bam` & `Reference.fa` | `Sample_library_Lane_RX_Ye.realignment_targets.list` & `Sample_library_Lane_RX_Ye.realigned.bam` | RealignerTargetCreator ID's targets, IndelRealigner realigns
5. | | `base_score_recalibration_array.qsub` | GATK/3.4.46 | '--run_without_dbsnp_potentially_ruining_quality' | `Sample_library_Lane_RX_Ye.realigned.bam` & `Reference.fa` | `Sample_library_Lane_RX_Ye.recal_data.table` & `Sample_library_Lane_RX_Ye.recal_plots.pdf` & `Sample_library_Lane_RX_Ye.recalibrated.bam` | 2 passes with BaseRecalibrator (with the infamous no-dbsnp flag), then AnalyzeCovariates prints the plots/stats, then PrintReads writes the output bam
6. | **Variant Calling** | `vcf_discovery_array.qsub` | GATK/3.4.46 | `--genotyping_mode DISCOVERY -stand_emit_conf 10 -stand_call_conf 30` | `Sample_library_Lane_RX_Ye.recalibrated.bam` & `Reference.fa` | `Sample_library_Lane_RX_Ye.raw_variants.vcf` | HaplotypeCaller does what it says on the tin
7. | | `compress_vcfs.qsub` | tabix/0.2.6 & vcftools/4.2 | | `Sample_library_Lane_RX_Ye.raw_variants.vcf` | `Sample_library_Lane_RX_Ye.raw_variants.vcf.gz` | each vcf file is compressed (for vcftools compatibility) and then tabix-indexed
8. | | `vcf_merge_samples_array.qsub` | tabix/0.2.6 & vcftools/4.2 | `--remove-duplicates` | `Sample_library_Lane_RX_Ye.raw_variants.vcf.gz` | `Sample_all_libraries.vcf.gz` | vcf files merged at the level of population
9. | | `merge_all_vcfs.qsub` | tabix/0.2.6 & vcftools/4.2 | | `Sample_all_libraries.vcf.gz` | `all_variants_merged_$date$.vcf.gz` | this is a long step; merging all the sample-level vcfs into one file for passing to SKAT
10. | **Statistical Analysis** | `calc_Fst_array` | vcftools/4.2 | `--fst-window-size` & `--fst-window-step` | `all_variants_merged_${date}.vcf` | `Fst_POP1_vs_POP2.windowedXkb.stepYkb.weir.fst` | Fst stat. calculated for all pairwise between-pop comparisons. Options passed into output filenames.


plink? SKAT?