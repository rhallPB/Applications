## Sample and Sequencing related
### Magbead
A sample loading method that increases loading preference for longer molecules. Highly recommended for transcriptome sequencing.

### Size Selection
Separation of library inserts by length before sequencing. Can be done with agarose gel cutting, BluePippin, or AMPure. The benefit of doing this is enabling capture of longer transcripts (> 2k) that are less abundant and less preferentially bound for sequencing. Typical size selection ranges are: < 1k, 1-2k, 2-3k, > 3k.

### StageStart
Enabling the collection of sequencing information as soon as the polymerase starts. Has the dual advantage of: (1) increasing read length, (2) increase the chance that the first-pass subread in transcriptome sequencing is full-length. Highly recommended for transcriptome sequencing.

Related web posts: [Comparison of read length w/ and w/out StageStart, U of Maryland](http://www.igs.umaryland.edu/labs/grc/2013/02/04/the-pacbio-stage-start-feature/)

## All those @#$% reads!
### raw read, subread, and CCS read
Read [here](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/Understanding-PacBio-transcriptome-data#wiki-readexplained) for detailed explanation.

<a name="fullpass"/>
### full-pass subread
A subread in which SMRTbell adapters were observed at both ends (but of course, were trimmed away during primary analysis). A candidate for a full-length transcript. This subread would still potentially contain 5' and 3' primers from the cDNA library kit. For runs using StageStart, the first-pass subread will often be considered NOT a full-pass subread by primary analysis due to incomplete SMRTbell adapter in the beginning. Hence for full-length identification, I recommend using [additional full-length detection scripts](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/Identifying-full-length-transcripts) to identify and trim away cDNA library primers.

<a name="53seen"/>
### 5'-3' subread aka putative full-length subread [UNOFFICIAL]
A temporary term for subreads identified by the [full-length detection scripts](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/Identifying-full-length-transcripts) as having both 5' and 3' primers from the cDNA library kit.

## Aligners

<a name="blasr"></a>
### BLASR
PacBio's in-house aligner. Designed to handle PacBio's read length and accuracy. Takes advantage of multicores, very fast and efficient. Useful for mapping reads to genomes or transcript sequences. However, it cannot handle splicing. For mapping spliced transcripts to genomes, use GMAP (officially supported in SMRTPortal 1.4, protocol RS_cDNA_Mapping). 

Publication: ["Mapping single molecule sequencing reads using basic local alignment with successive refinement (BLASR): application and theory", M Chaisson and G Tesler, _BMC Bioinformatics_, 2012](http://www.biomedcentral.com/1471-2105/13/238)

<a name="gmap"/>
### GMAP
Aligner for mapping spliced transcripts to genomes. Originally developed for mapping ESTs and Sanger sequences, thus can handle PacBio long reads very well. Now supported in SMRTPortal 1.4 (protocol RS_cDNA_Mapping). 

Official website: http://research-pub.gene.com/gmap/

