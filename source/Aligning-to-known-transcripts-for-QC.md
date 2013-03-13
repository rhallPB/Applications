1. [Purpose](#purpose)
2. [Prerequisites](#prereq)
3. [Step 1: Setting up & running SMRTPortal](#step1)
4. [Step 2: Running the full-length identification script](#step2)
5. [Step 3: Running alignQC.py](#step3)
6. [Step 4: Understanding the alignQC output](#step4)
7. [Final Word on alignQC's purpose](#finalword)

***

<a name="purpose"/>
## Purpose
The purpose of this page is to provide a walkthrough of using SMRTPortal to filter and align (using BLASR) subreads to a known transcript database (such as [Gencode](http://www.gencodegenes.org/) for human) and then plot various figures from the alignment output using the [alignQC.py](https://github.com/Magdoll/cDNA_primer/blob/master/scripts/alignQC.py) script in the [cDNA_primer](https://github.com/Magdoll/cDNA_primer) repository. It is more for quality control than actual analysis. I have been using alignQC.py as a means to:

* Plot subread lengths. If size selection was done on the libraries, validate that the full-length (FL) subread lengths agree with the size selection range.
* Plot 5'-3' coverage on aligned reference transcripts. Although there might be novel isoforms, for well-annotated organisms such as human, I would expect that the majority of FL subreads would align to the entire reference transcript. 
* Plot relation between subread and reference length. 

The protocol and scripts on this page are currently **UNOFFICIAL**. The currently official transcriptome-related protocol, RS_cDNA_Mapping, maps subreads to genomes using [GMAP](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/Glossary-for-PacBio-transcriptome#gmap), which is different from this tutorial's protocol, which maps subreads to transcripts using PacBio's own aligner [BLASR](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/Glossary-for-PacBio-transcriptome#blasr). Hopefully this will eventually get integrated into the official SMRTPortal...

The majority of the plots use only putative FL subreads based on the output from my FL-identification scripts. Read [here](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/Identifying-full-length-transcripts) on how to run the FL scripts.

<a name="prereq"/>
## Prerequisite: SMRTPortal
I have tested this on SMRTPortal 1.4, but it should work for 1.3 as well. Support for SMRTPortal 2.x will come soon. Also, the protocols that I use are unofficial. I modified them for transcriptome analysis. You will need to be able to copy-paste a protocol .xml file directly into the smrtanalysis directory from the filesystem side (which probably means you need root/admin permission...)

## Prerequisite: Python libraries
* Python 2.7
* [BioPython](http://biopython.org/wiki/Main_Page)
* [numpy/scipy](http://www.scipy.org/)
* [bx-python](https://bitbucket.org/james_taylor/bx-python/wiki/Home)
* pbcore (see footnote)

In addition, to run the full-length identification script you need:
* [PBBarcode](https://dl.dropbox.com/u/47842021/wiki_transcriptome/soft/PacBioBarcodeCCSDist.zip)
* [HMMER](http://hmmer.janelia.org/software/archive) (must get HMMER2; HMMER3 won't work). 

FOOTNOTE: if you have SMRTPortal installed you should have pbcore library. In python, if you do:

``import pbcore``

and it succeeds. Then nothing else needs to be done. If not, you need to find where the pbcore library lives and add it to your PYTHONPATH. 

<a name="step1"/>
## Step 1: Setting up & running SMRTPortal
### Copying the protocols
The protocols you need are [here](https://github.com/Magdoll/cDNA_primer/tree/master/protocols). You will need to know where the smrtanalysis protocols live. Most likely the directory path is something like _/XXXX/opt/smrtanalysis/common/protocols_. Copy [transcript_gencode13.3.xml](https://github.com/Magdoll/cDNA_primer/blob/master/protocols/transcript_gencode13.3.xml) to the _protocols/_ directory, and [BLASR_transcript.1.xml](https://github.com/Magdoll/cDNA_primer/blob/master/protocols/BLASR_transcript.1.xml) to the _protocols/mapping_ subdirectory.

If all goes well, you should see the protocol show up in SMRTPortal when you go to the Manage Protocol (Home->Import and Manage->Manage Protocols) page:
![Protocol Screenshot](https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/step0_protocol.png)

### Setting up the reference
You can add new reference sequences through the SMRTPortal interface (Home->Import and Manage->Manage Reference Sequences). I don't know how it works at other places, but the way I do it is first copy the fasta file to _/XXXX/opt/smrtanalysis/common/references_dropbox_, which then will show up in the SMRTPortal Manage Reference Sequences page so I can add it.

![AddRef Screenshot]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/step0_addref.png)

For the human Gencode, I have provided the [non-redundant Gencode v13 fasta file](https://dl.dropbox.com/u/47842021/wiki_transcriptome/gencode/gencode.v13.pc_transcripts.non_redundant.fa) that you can download. If you are using other reference database, beware that the reference import will FAIL if there are duplicate sequences in the fasta file. In the case of Gencode, there were multiple identical sequences with different IDs and I had to manually remove them.

If you're downloading my Gencode fasta file, also download the [strand information file](https://dl.dropbox.com/u/47842021/wiki_transcriptome/gencode/gencode.v13.annotation.gtf.info.pickle) (packaged in a Python pickle). You will need this later in alignQC.py to get proper strand information for plotting 5'-3' reference transcript coverage.

### Setting up the job
Setting up the job through SMRTPortal is the same as any other jobs. In this test case, I'm only putting 1 SMRT cell in and am using Gencode v13 as reference. If you're using a different reference, just click on the Reference drop-down menu and select the appropriate one.

![Job Screenshot]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/step0_design.png)


### A word about multiple isoforms or highly similar genes
Part of the tricky thing about aligning subreads to transcripts using BLASR is that: (1) if there are multiple isoforms or highly similar sequences in the DB, the default behavior for BLASR is to randomly align to one of them and report and (2) BLASR can output partial matches (ex: only 30% of the subread is aligned to some transcript). This is because BLASR is designed to be as flexible and error-tolerant as possible.

Currently, to minimize potential bias in the alignQC.py results due to these two problems, the (somewhat ad-hoc) solution is to (1) increase the number of hits to report in BLASR to 10 and (2) in alignQC.py, if a subread maps to more than one transcript, select the one for which the coverage on the transcript is highest. If you want to crank up the number of hits BLASR reports, you can change the value of _maxHits_ in _transcript_gencode13.3.xml_.


### Wait for the job to finish...kind of
When the job is done, it will say "Completed". An indication that the data we need is complete can be found in the lower left corner of the job status page. 

![JobDone Screenshot]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/step1_jobdone.png)


To continue on, we need the following files: _filtered_subreads.fasta_ and _aligned_reads.cmp.h5_. You can download them by clicking on the link, OR, here's the hacky but awesome way to find out where the files are on the linux filesystem. Click on "View Log" and scroll through the log to find indication of where the data directories have been created. In this case, the path is ``/net/usmp-data3-10g/ifs/data/vol53/fas/secondary/smrtanalysis-1.4.0/common/jobs/016/016840``.

![FindPath ScreenShot]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/step2_findpath.png)

In the job directory, there is a subdirectory called ``data``. Here you will find both _filtered_subreads.fasta_ and _aligned_reads.cmp.h5_. Often times I just [symbolically link](http://en.wikipedia.org/wiki/Symbolic_link) to them and save myself the trouble of copying!


<a name="step2"/>
## Step 2: Running the full-length identification script
alignQC.py relies on the .primer_info.txt file generated from my FL identification scripts. Refer to [this wiki](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/Identifying-full-length-transcripts) to extract FL information from _filtered_subreads.fasta_. When it's done you should have a file called _filtered_subreads.53seen_trimmed.fa.primer_info.txt_.

<a name="step3"/>
## Step 3: Running alignQC.py
### Optional (but recommended): setting up reference transcript strand information
There are only 1 set of figures (the 5'-3' reference coverage plot) that are affected if you don't have strand information. Hence it is optional, but if you want to use strand information, just make a Python pickle of a dictionary object where the key is the reference sequence ID and the value is a dictionary with at the very least, the key 'strand'.

Here's an example dictionary (actually, a dict of dict...):
``d={'ENST00000459668.1':
 {'chr': 'chr10',
  'gid': 'ENSG00000175029.11',
  'status': 'KNOWN',
  'strand': '-',
  'type': 'protein_coding'}}``

Just pickle it and you're good.


### Running alignQC.py (finally!)
The basic usage of alignQC.py is:

``alignQC.py -d <output_directory> -p <output_prefix> -m <primer_info_txt> <smrtportal job directory>``

If you have reference strand information, use ``----refStrandPickle <pickle_filename>`` and if you did size selection and want to restrict to looking at reference transcripts within a certain length range, use ``--ref_size "(min_length,max_length)"``. 

For example, this is command used for the test job above:

> alignQC.py -d outputQC -p wikitest
> -m primer_match/wikitest/filtered_subreads.53seen_trimmed.fa.primer_info.txt 
> --refStrandPickle gencode.v13.annotation.gtf.info.pickle
> /net/usmp-data3-10g/ifs/data/vol53/fas/secondary/smrtanalysis-1.4.0/common/jobs/016/016840

Depending on how many SMRTcells you ran and how big alignment file is, the script could take a while.

When all is completed, you should get 3 PDF files and 1 Python pickle (.pkl) in the output directory. The pickle can be read back by alignQC.py if for some reason you want to re-plot the PDFs by using the ``--read_pickle`` option (mostly this was for me to quickly change the code and re-run the script without having the parse the .bas.h5 and .cmp.h5 files again). The _output_prefix_.summary.pdf and _output_prefix_.figures.pdf are combined to create _output_prefix_.pdf.

<a name="step4"/>
## Step 4: Understanding the alignQC output

The output from running the test dataset above is [wikitest.pdf](https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/wikitest.pdf). I'll go through the figures one by one.

### Summary stats (p.1)

The first page is summary stats. You see where the source directory was, how may sequencing ZMWs there were, how many subreads you got, what percentage of the subreads (or ZMWs) had both 5' and 3' primers seen, etc.

![Summary stats]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/alignQC_summary.png)

### Histogram of subread length (p.2)

Length distribution of all subreads, just [full-pass subreads](https://github.com/Magdoll/cDNA_primer/wiki/The-unofficial-glossary-of-terms-for-PacBio-transcriptome#fullpass), and just [5'-3' subreads](https://github.com/Magdoll/cDNA_primer/wiki/The-unofficial-glossary-of-terms-for-PacBio-transcriptome#53seen). I tend to see two peaks for all subreads, one for the full-length ones, one for the non-full-length ones. The full-pass and 5'-3' distribution should generally be very similar and the peak range should correspond to the library size selection if it was done. In this case, there was no size selection, so the distribution is more widespread.

![SubRL]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/alignQC_subRL.png)


### Histogram of aligned subread proportion (p.3)

A tally of the aligned proportion of each subread. Later in the PDFs I am calling this "qCov". So if a subread is fully aligned to a transcript, then qCov=1. If the transcript database is exhaustive and there are no subreads coming from novel transcripts, then the histogram should peak at 1. However, it is not uncommon to see alignments with qCov << 1 because the database is not exhaustive and novel isoforms are found.

![Subread Aligned Histogram]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/alignQC_subAlnHist.png)


### Histogram of aligned reference proportion (p.4)

Like p.3, but for reference transcript lengths. "All references" plots the length distribution of everything that's in the database. In this case, Gencode has a lot of sequences under 1kb. For the other two curves, only references for which at least one subread aligned to it with qCov>=80% is plotted. 

![Ref Aligned Histogram]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/alignQC_refAlnHist.png)


### 5'-3' coverage of aligned references (p.5-8)

p.5 & 7 is plotted for full-pass, and p.6 & 8 for 5-'3' subreads. For each category of subreads, all alignments with qCov>=80% is used, and the start and end position of the alignment on the *reference* is normalized by the reference length. So if the entire reference transcript is aligned, the start would be 0 and the end would be 1. And if all alignments are like this, the lower half of the plot would be a rectangle. The plot below shows decent and uniform 5'-3' coverage. 

Note that this is the only two plots in the PDF that uses the strand information pickle. p.7-8 are the same as p.5-6 except that only one datapoint is used per reference.

![5'-3' Ref Coverage]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/alignQC_53coverage.png)

### Abundance and length of aligned references (p.9-10)

The X-axis is the reference length. The Y-axis is the number of subreads that hit a reference at this reference length range. The size of the circle is how many reference genes were in that (x,y) datapoint.

p.9 is plotted for aligned references with qCov>=80%; p.10 is for qCov>=10%.

![Ref Abundance]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/alignQC_abundance.png)


### Aligned subread length VS reference length (p.11-12)

For a well-annotated database like Gencode, we expect that most well-aligned (qCov>=80%) full-length subread lengths will be the same as the reference length. The plot below shows that most datapoints are on the 45 degree line. 

![Subread Length vs Reference Length]
(https://dl.dropbox.com/u/47842021/wiki_transcriptome/smrtportal_screenshots/alignQC_subVSref.png)

p.13-16 are all very similar to the way p.11-12 is plotted, just with the variables changed a bit. The figure titles should be self-explanatory.

<a name="finalword"/>
## Final Word on alignQC's purpose

By now it should be clear what alignQC is not doing: it is not meant for discovering novel transcripts or isoforms. In fact, these get *excluded* from the plots. It is really used for a **sanity check**. Use it to see that you are recovering known transcripts (which is always a good sign), that subread lengths agree with your library size selection and aligned reference lengths, that the particular cDNA kit you're using is not causing strong 5' and 3' end bias, etc.
 
Finally, while the script has gone through many iterations, it is always possible that there are BUGs. If anything looks odd or you think adding another plot would make it even more useful, shoot me an [email](mailto:etseng@pacificbiosciences.com).