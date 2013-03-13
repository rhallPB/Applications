1. <a href="#background">Background information</a>
2. <a href="#readexplained">Raw read, subread, CCS and what they mean for transcriptome data</a>
3. <a href="#getFL">How to determine if a subread (or CCS read) is a full-length transcript</a>
4. <a href="#short_errcor">Using Illumina short reads to improve base accuracy</a>
5. <a href="#flowchart">Summary: Flowchart for transcriptome analysis</a>


<a name="background"></a>
## Background Information
Below are some background information you might find helpful if you are completely unfamiliar with SMRT Sequencing or want to know about how to do sample preparation for full-length cDNA sequencing.

* Webinar links to: [SMRT Sequencing Technology](http://aa314.o1.gondor.io/webinar/smrt-sequencing-technology-overview/), [explanation of data](http://aa314.o1.gondor.io/webinar/specifics-of-smrt-sequencing-data/) (difference between raw read, subread, or CCS read?), [primary analysis](http://aa314.o1.gondor.io/webinar/primary-analysis/) (base calling and QVs), and [full-length cDNA sample preparation](http://aa314.o1.gondor.io/webinar/sequencing-full-length-cdna/)

* Sample preparation: [Full-length cDNA sample protocol](http://www.smrtcommunity.com/Share/Protocol?id=a1q70000000H6MLAA0&strRecordTypeName=Protocol) (registration required)

* [Official Glossary of Terms](http://www.smrtcommunity.com/SampleNet/Sample-Prep) is available on SampleNet (registration required)

* [Primary Analysis output (.trc.h5, .pls.h5, .bas.h5) explanation](http://www.smrtcommunity.com/SMRT-Analysis/Algorithms/Primary-Analysis)

* [Secondary Analysis through SMRT portal](http://aa314.o1.gondor.io/webinar/secondary-analysis/)

<a name="readexplained"></a>
## Raw read, subread, CCS and what they mean for transcriptome data
**NOTE: all analysis shown below is based on SMRT Pipe version 1.4**

At the end of a sequencing run, you'll get .bas.h5 files in the Analysis_Results/ subfolder.

![bash5_folder.png](https://dl.dropbox.com/u/47842021/wiki_transcriptome/bash5_folder.png)

The [.bas.h5](http://pacificbiosciences.com/devnet/files/software/primary-analysis/1.2.2/doc/Base,%20Pulse,%20and%20Trace%20File%20Reference%20Guide.pdf) file contains the raw base calls along with quality metrics (QVs). Unless you need access to quality values (Phred scores indicating how likely the base is a mismatch/substitution, insertion, deletion, etc), you should not have to deal with .bas.h5 directly.

_m121208_091821_42175_c000441702559900001500000112311424_s1_p0_ is the movieName. If there are multiple movies in the same run, they are distinguished by the prefix "s", so the second and third movie would have the prefix:

_m121208_091821_42175_c000441702559900001500000112311424_s2_p0_
_m121208_091821_42175_c000441702559900001500000112311424_s3_p0_

and so on.

_m121208_091821_42175_c000441702559900001500000112311424_s1_p0_.fasta contains the raw read fasta sequences. If there are 75k [ZMWs](http://www.pacificbiosciences.com/products/smrt-technology/) in a SMRT Cell, then there would be (more or less) 75k sequences in the raw read fasta file. The raw read fasta file should not be used directly for transcriptome analysis because it does not represent the single sequence of a transcript. Rather it is the sequencing result of going around and around the circular SMRTbell structure. Below is a toy example of a raw sequence:

![raw sequence unrolled](https://dl.dropbox.com/u/47842021/wiki_transcriptome/rawread_unrolled.png)

It is [Secondary Analysis](http://www.pacificbiosciences.com/products/pacificbio-rs-workflow/#tab-6) that will do the job of removing the SMRTbell adapters, trimming away low quality regions, and breaking the raw read up into (in this case, three) separate **subreads**.

You can run Secondary Analysis either through the SMRT Portal interface or through command line (smrtpipe.py). Either way, in the output data/ subdirectory you will see:

![smrtpipe output data subfolder](https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtpipe_output_data.png)

_filtered_subreads.fasta_ is the filtered subreads. The sequence ID are named by:

``<movieName>/<ZMW number>/<subread start_subread end>``

The subread start is the 0-based, end is 1-based (same as used by [UCSC](http://genome.ucsc.edu/FAQ/FAQtracks.html#tracks1)) offset on the raw read. 

Going back to the toy example above, suppose the raw read was 2,850 bp. The first "pass" of the sequencing (before reaching the adapter the first time) went on for 1000 bp. The next 50bp is the adapter sequence. Then it goes to the other side of the SMRTbell and sequencing the transcript again, this time in reverse complement (which is why I'm showing the poly A tail is a series of 'T's), then at 2,100 it hits the adapter again, and goes on to its third pass of sequencing, but alas, time ran out or the polymerase got tired and stopped there.

The corresponding subreads would have the sequence IDs:

``m120530_022750_42142_c100391630010000001523040611021240_s2_p0/14/0_1000``
``m120530_022750_42142_c100391630010000001523040611021240_s2_p0/14/1050_2100``
``m120530_022750_42142_c100391630010000001523040611021240_s2_p0/14/2150_2850``

If you are counting transcripts, it is important to count by ZMW, and not by subread. All subreads from the same ZMW are coming from the same transcript. 

Now I can talk about **CCS reads** (Circular Consensus Sequencing). From Secondary Analysis output you'll get a file called _filtered_CCS_subreads.fasta_. The sequence ID naming convention is:

``<movieName>/<ZMW number>``

So it only has the movieName and ZMW number. That's because, since we know that every subread from the same ZMW is just sequencing the same molecule(but with random sequencing errors), we can align them and output one consensus sequence. That's what CCS reads are. 

Not all ZMWs get a corresponding CCS read. Right now, the criterion is that a ZMW has to have at least two "full-pass" subreads (full-pass defined by being flanked on both sides by adapters). In the toy example above, because only the second pass is a full-pass subread, it does not get a CCS.

If your transcript lengths are short (1-2k), chances are a good number if not most of the ZMWs will get a CCS. In that case you can probably just work on the _filtered_CCS_subreads.fasta_ file. CCS has a higher accuracy (> 95%) than raw read accuracy (~85%), and can be easily mapped to known transcripts or genomes using alignment tools such as BLAST, [BLAT](http://genome.ucsc.edu/cgi-bin/hgBlat?command=start), [GMAP](http://research-pub.gene.com/gmap/), or PacBio's own [BLASR](https://github.com/PacificBiosciences/blasr, see NOTE2). In my experience, at ~85% single pass accuracy, the subreads can also be mapped with BLAST, BLAT, and GMAP, but occasionally it would require a bit more post-processing. BLASR is designed to work with PacBio's read lengths and single- and multi-pass accuracy, so it will work well on both CCS and subreads.

**NOTE:** currently, if a ZMW gets a CCS read, it is output both as single CCS read in _filtered_CCS_subreads.fasta_ and as separate subreads in _filtered_subreads.fasta_. If you want to create a subread file that is exclusively non-CCS, you can use [this script](https://github.com/Magdoll/cDNA_primer/blob/master/scripts/grab_nonCCS_subreads.py) from a [cDNA project](https://github.com/Magdoll/cDNA_primer) on GitHub.

**NOTE2:** In the case of splicing, BLASR is recommended only for mapping subreads or CCS reads to transcript sequences, not to a genomic reference. This is because BLASR is not splicing-aware; it will simply treat short introns as long deletions and will fail to detect distant junctions. GMAP is designed to work with both spliced and non-spliced transcripts. BLAT, although not originally designed for splice mapping, has shown in practice to be capable of mapping spliced transcripts back to the genome, though it usually requires extra output processing. 

<a name="getFL"></a>
## How to determine if a subread (or CCS read) is a full-length transcript

With PacBio's long readlength (currently, mean around 3k, with the longest going beyond 10k), most transcripts have a chance of being sequenced in full-length. No assembly would be required.

But going back to the toy example again:

![raw sequence unrolled](https://dl.dropbox.com/u/47842021/wiki_transcriptome/rawread_unrolled.png)

Clearly, not all subreads are full-length (FL). CCS reads, on the other hand, are almost guaranteed to be FL transcripts because of the stringent criterion of seeing two or more full-passes (from a fairly large sampling of data, >99% of CCS reads pass the full-length test). If you only work with CCS reads, then you can assume all of them to be full-length and apply them directly for alignment or further analysis. Just remember that your 5' and 3' primers (from the cDNA prep step) will be flanking the transcript. In fact, one way to determine that transcripts are full-length is to use the 5'/3' primer information from the cDNA kit. Seeing both the 5' and 3' primer and maybe even a poly A tail, is the best guarantee of a FL transcript.

A side benefit of using 5' and 3' primer for FL identification is that, if the primers are not symmetric, the orientation of the sequence (in the absence of poly A tail) can be determined. In the toy example case, by seeing the reverse complement of the 3' primer, followed by poly Ts, and ending with the reverse complement of the 5' primer, we can tell this is the reverse complement of the actual transcript.

A [script for identifying and trimming 5'/3' primers](https://github.com/Magdoll/cDNA_primer) are available on GitHub. The script provides a clean output of subreads (or CCS reads) that are potentially full-length transcripts.

<a name="short_errcor"></a>
## Using Illumina short reads to improve base accuracy

[PacBioToCA](http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=PacBioToCA) and [LSC](http://www.stanford.edu/~kinfai/LSC/LSC.html) are two available tools developed outside PacBio for using Illumina short reads to improve base accuracy. Refer to [this wiki page](https://github.com/Magdoll/cDNA_primer/wiki/Error-Correction-using-Illumina-short-reads) for further details.

<a name="flowchart"></a>
## Flowchart for transcriptome analysis

Below is a recommended flowchart for analyzing PacBio transcriptome data.

![Flowchart](https://dl.dropbox.com/u/47842021/wiki_transcriptome/myflowchart.png)