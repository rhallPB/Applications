1. <a href="#ca">Using PacBioToCA for error correction</a>
2. <a href="#lsc">Using LSC for error correction</a>

***

<a name="ca"></a>
## PacBioToCA

[PacBioToCA](http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=PacBioToCA) can be use to error correct PacBio long reads using short reads such as Illumina.

Publication: [**Hybrid error correction and de novo assembly of single-molecule sequencing reads**, Koren et al., _Nat Biotechnol._, 2012](http://www.nature.com/nbt/journal/v30/n7/full/nbt.2280.html)

For the latest information, always follow the [official wiki](http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=PacBioToCA).

General guideline for using pacBioToCA from [PacBio's own wiki](https://github.com/PacificBiosciences/Bioinformatics-Training/wiki/pacBioToCA). While this page is useful the parameters are not specific for transcriptome data. See below.


### Recommended steps and parameters for transcriptome data
The following recommendations are provided by S Koren:

The best version of the code to use is from CVS as of July 13th, 2012 (after that one of the CA developers accidentally broke the correction pipeline). Some of the parameters below were not exposed in the pre-built version (CA 7.0). You can get the code by running the command:

``cvs -z3 -d:pserver:anonymous@wgs-assembler.cvs.sourceforge.net:/cvsroot/wgs-assembler co -D "2012-07-13" -P wgs-assembler``

To build it you can [follow the instructions here](http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=Check_out_and_Compile).

You also need the kmer package in order to build it.

Also, in _AS_global.h_ you have to change <br>
** #define AS_READ_MAX_NORMAL_LEN_BITS 11 **<br>
to<br>
** #define AS_READ_MAX_NORMAL_LEN_BITS 15**<br>

As far as the parameters, there are two you want to adjust for the coverage:

- The first parameter, _ovlMerThreshold tells_ the correction script which kmers are too repetitive to be seeds. By default, it will try to pick non-repetitive ones to speed up the alignment. However, in your case, there won't be a single coverage so you can manually set it to look at all kmers, even repetitive ones. Setting it to 0 will ensure it uses all kmers as seeds, which will slow the pipeline down, but make sure you do not miss any seeds. Alternatively, you can try setting it to between 5000-10000 which I would expect to cover most of your sequences and speed up the pipeline.

- The second parameter, _-coverage_, tells the correction how many matches each Illumina sequence is allowed to have. By default, it looks at the distribution and picks a threshold to eliminate repeat-induced matches. Again, I would expect setting it to 5000-10000 should be sufficient (but you can increase it up to 65,000). 

The parameters below are what we used for the parrot spec file (and replace the defaults posted [here](http://www.cbcb.umd.edu/software/PBcR/data/sampleData/pacbio.SGE.spec)). I included the _ovlMerThreshold_ parameter as well to make the correction pipeline examine all seeds:

>useGrid = 1<br>
>scriptOnGrid = 1<br>
>frgCorrOnGrid = 1<br>
>ovlCorrOnGrid = 1<br>
><br>
>sge = -A assembly<br>
>sgeScript = -pe threads 16<br>
>sgeConsensus = -pe threads 1<br>
>sgeOverlap = -pe threads 5<br>
>sgeFragmentCorrection = -pe threads 2<br>
>sgeOverlapCorrection = -pe threads 1<br>
><br>
>ovlMerThreshold=0<br>
>ovlHashBits = 25<br>
>ovlThreads = 5<br>
>ovlHashBlockLength = 1000000000<br>
>ovlRefBlockSize =  100000000<br>
>doOverlapBasedTrimming = 0<br>

This will make each overlapper job use approximately 18GB of RAM. Assuming you have 64GB of ram and 16 cores on your nodes, you can run three jobs per node and each will use 5 threads (based on the _ovlThreads_ parameter above). If you have less memory or more cores you can increase the number of threads each overlap job uses to fill the memory/cores of your machines. There is more information on how to [tune overlap parameters for Celera Assembler](http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=RunCA#OVL_Overlapper).

Finally, below is a brief into to the output log files.

If you have a recent version of the software, it generates a file named \<library name\>.log with the translation table:

>INPUT_NAME OUTPUT_NAME<br>
>SUBREAD START<br>
>END LENGTH<br>
>200000025859 pacbio_25859_1<br>
>1 62<br>
>780 718<br>
>200000025862 pacbio_25862_1<br>
>1 13<br>
>648 635<br>
>200000025864 pacbio_25864_1<br>
>1 6<br>
>804 798<br>
>200000025865 pacbio_25865_1<br>
>1 11<br>
>907 896<br>
>200000025866 pacbio_25866_1<br>
>1 1<br>
>526 525<br>
>200000025867 pacbio_25867_1<br>
>1 0<br>
>553 553<br>

The first column is the original name, the second column is the new name, the next column is the number of sequences that have come from this original PacBio read (how many pieces it was split into), and the final columns give the start and end positions of the corrected sequence within the original read. This code change was added on 9/13/2012. If your code predates this version, you have to go to a text file stored in the temporary correction directory. In the temp<libraryname> directory, there is a file named _asm.gkpStore.fastqUIDmap_:
>300003067117 3067117<br>
>m120722_174903_42196_c100366212550000001523028810101297_s1_p0/27<br>

The third column is the external sequence identifier. The middle column is the output ID. So for the line above, there will be corrected sequences named <libraryname>_ 3067117_<subID> where subID identifies how many sequences the original read (m120722_174903_42196_c100366212550000001523028810101297_s1_p0) was split into by correction. If you no longer have the temporary directory, you can re-create this file by running the commands:
``fastqToCA -libraryname PacBio -type sanger -innie -technology pacbio-long -reads <your pacbio fastq> > pacbio.frg``
``gatekeeper  -o asm.gkpStore  -F <your correction data>.frg pacbio.frg ``


<a name="lsc"></a>
## LSC

LSC is developed by KinFai Au at the Wong lab at Stanford. It uses homopolymer compression to correct long reads. 

For installation and usage, refer to the [official LSC page](http://www.stanford.edu/~kinfai/LSC/LSC.html).

Publication: [**Improving PacBio Long Read Accuracy by Short Read Alignment**, Au et al., _PLoS ONE_, 2012](http://www.plosone.org/article/info%3Adoi%2F10.1371%2Fjournal.pone.0046679)