# Barcoding

This page describes general guidelines and suggestions for using the barcoding protocol in SMRT® Analysis version 1.4.  The emphasis here is on the bioinformatics aspects of barcoding protocol, including some considerations for sample preparation of barcoded insert sequences.  More detail on sample preparation can be found at www.pacbiosamplenet.com.

## Introduction

SMRT® Analysis version 1.4 has added barcoding functionality which allows for detection and identification of unique barcodes and to subsequently:  

  1. Split reads into separate files per barcode
  2. Call variants on individual barcodes using GATK Unified Genotyper

The SMRT® Portal protocol for barcoding is named RS_Resequencing_GATK_Barcode.  As of SMRT® Analysis version 1.4 barcoding is only supported for analysis of multiple pass (CCS) inserts.

## Barcode Sequences

A set of 48 pairs of 16bp barcodes custom designed for the PacBio RS error mode is recommended for use in SMRT® barcoding protocols. This set of barcodes was designed with the following properties:

* Uniquely distinguishable in both forward and reverse complement orders
* Optimized for distinguishing between pairs
* Numerically ordered starting with the most differentiable barcodes

Links to downloadable files for ordering primers with the recommended barcodes and the corresponding barcode FASTA files for analysis in SMRT® Portal can be found below.  To use only a subset of the barcodes provided, simply select the desired barcodes and also truncate the FASTA file to reflect the barcodes present in the sample set.   


* [Guidelines for ordering primers with barcodes](http://www.smrtcommunity.com/Share/Protocol?id=a1q70000000HC3cAAG&strRecordTypeName=Protocol)
* [FASTA for analysis with barcodes](http://pacb.com/devnet/training/pacbio_barcodes_paired.fasta)

###Padding Sequence

In order to help normalize ligation efficiencies between SMRTbell™ adapters and barcodes across all sequences, it may be beneficial to include a short (5bp) constant padding.  It is important to place the padding sequence in the correct relative order with respect to the barcode and insert, both for ordering primers and barcode identification during analysis.  In the following examples, the constant padding sequence is always upper case (GGTAG), while unique barcode sequences are lower case (gcg...).  Note that the padding is reversed in the R1 barcode FASTA but not complemented.  
    
FASTA sequence examples:

    >F1 
    GGTAGgcgctctgtgtgcagc
    >R1
    agagtactacatatgaGATGG

Primer sequence examples:

    F1 Barcode Forward Primer: GGTAGgcgctctgtgtgcagcNNNNNNNNNNN (“N” = target forward primer)
    R1 Barcode Reverse Primer: CCATCtcatatgtagtactctNNNNNNNNNNN (“N” = target reverse primer)
    
With the above primer design, a template _with padding_ would have the following structure:

    Adapter – Padding Sequence – Forward Barcode – Forward Primer – INSERT – Reverse Primer – Reverse Barcode – Reverse Padding Sequence – Adapter

###Custom Barcode Sequences

Users designing their own barcodes should be aware of the PacBio RS error mode, in particular, homopolymer sequences should be avoided.  For correct identification of barcodes during analysis, be sure to include all sequence between the SMRTbell™ adapter and the insert side end of the barcode sequence in the FASTA file which is passed to SMRT® Portal.               

## Scoring Modes

SMRT® Analysis v1.4 barcoding protocol includes three different scoring modes which can be used according to study design and needs.

### Paired Barcodes

In this model a pair of barcodes always occurs together, one on each end of the insert, each having a different sequence.  Paired barcodes should be listed in pairwise order in the barcode FASTA file.

    >F1
    GGTAGgcgctctgtgtgcagc
    >R1
    agagtactacatatgaGATGG
    >F2
    GGTAGtcatgagtcgacacta
    >R2
    cgtgtgcatagatcgcGATGG
    ...

In the above case, F1 is paired with R1 (F1--R1), F2 is paired with R2 (F2--R2), and so on.  In SMRT® Analysis v1.4 this scoring mode is only recommended for insert sequences of <3kb such that _both_ barcodes are expected to be read for any single molecule, as final output label will depend on the highest scoring _pair_ of barcodes.

### Symmetric Barcodes

This model assumes the same barcode sequence is appended to both ends of an insert sequence, e.g. F1--RC(F1).  Symmetric mode is not recommended when target sequences are amplified using PCR due to possible hairpins caused by the barcode sequence.  

### Asymmetric Barcodes

In the asymmetric mode, inserts are assumed to have different barcode sequences on either end and *barcodes can occur in any combination*.  Output barcode labels are determined by the two (2) best aligned barcodes.  As with the paired mode, the asymmetric mode is recommended for use with inserts <3kb such that *both* ends 
of the insert are expected to be read.  For ease of analysis and ordering barcoded primers, we recommend using one 'F' barcode and one 'R' barcode from the barcode list provided above for use in asymmetric mode.  In this use case, the template structure for asymmetric mode will have the same general structure as described above. 

## SMRT® Analysis Barcode Inputs

Setting up a SMRT® Portal barcoding job has only three barcode-specific parameters that need to be set after selecting the appropriate SMRT® Cells containing your barcoded data and selecting RS_Resequencing_GATK_Barcode protocol.  The first two can be found at the bottom of the Mapping pane of the Protocol Details dialog (Fig. 1).

![Figure 1.](images/barcode_protocol_details.png)

### Barcode Structure

Select from the barcode modes described above.

### Barcode FASTA file

The default is pre-set to point to the 48 paired barcodes _with padding_ in the reference directory.  User-defined FASTA files of barcode sequences do not need to be imported to the reference database as long as the file is stored in a location accessible by the SMRT® Portal installation.

### GATK Maximum coverage

The third parameter is the Maximum Coverage value for the GATK Unified Genotyper, found in the Consensus pane of the Protocol Details dialog.  This value is per barcode per amplicon, so for large datasets with many barcodes it may be necessary to decrease this number.  For most applications, the default should be ok.

## SMRT® Analysis Barcode Outputs

In addition to the typical Resequencing outputs, the RS_Resequencing_GATK_Barcode protocol produces two additional data products that can be downloaded from the Data panel in SMRT® Portal.

1. Barcode Fastqs:
   A compressed file containing FASTQ files of *subreads* for each barcode.
2. Variants (VCF,GFF):
   The output from GATK Unified Genotyper, with each barcode identified as a separate read group.  Results are reported on the basis of *subread* analysis.  A good place to find more information on how to interpret .vcf outputs is [here](http://gatkforums.broadinstitute.org/discussion/1268/how-should-i-interpret-vcf-files-produced-by-the-gatk).

## Command-line tools

Users wishing to retrieve barcoded CCS reads from a barcoded resequencing job can use the command line tool `pbbarcode.py`.  

    pbbarcode.py emitFastqs input.fofn barcode.fofn

Where the .fofn files are found in the job directory.  Additional options are listed by using `pbbarcode.py emitFastqs --help`.