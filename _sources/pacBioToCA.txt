pacBioToCA (error correction via Celera Assembler)
==================================================

PacBioToCA is a module in the Celera Assembler software package that performs error correction on PacBio long reads by mapping shorter, high accuracy reads onto the long reads. The error corrected reads can then be assembled using a long-read assembler, such as Celera Assembler, Mira, or Allora.

Installing and using PacBioToCA and Celera Assembler with PacBio data is documented extensively on the `pacBioToCA wiki page`_ (and the software is bundled in SMRT Analysis), so this page will serve as a brief tutorial.

The following hybrid assembly of lambda sequence uses two fastq files, one containing long reads, another containing short reads (e.g. Illumina, 454, CCS). For example data, please see the `pacBioToCA wiki page`_ under the "Example with Phage Reads" section.

.. _PacBioToCA wiki page: http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=PacBioToCA

Preparing the short read data
-----------------------------

First, create a frg file based on the short read fastq data, which contains metadata about the data (technology, whether it's paired end, etc.)::

    /tools/wgs-7.0/Linux-amd64/bin/fastqToCA -libraryname illumina -technology illumina -reads illumina.fastq > illumina.frg

Some points to consider:

* The ``-libraryname`` is required, but isn't important for any downstream analysis.
* Verify that the QV values are offset 33; otherwise use the -type parameter.
* If paired end data, make sure ``-insertsize`` and ``-innie``/``-outie`` are correctly specified.
* The frg file contains the absolute path of the fastq, so moving either will corrupt the frg file.

The documentation is presented in full in the following::

    usage: /tools/wgs-7.0/Linux-amd64/bin/fastqToCA [-insertsize <mean> <stddev>] [-libraryname <name>]

      -insertsize i d    Mates are on average i +- d bp apart.

      -libraryname n     The UID of the library these reads are added to.

      -technology p      What instrument were these reads generated on ('illumina' is the default):
                           'sanger'   --
                           '454'      --
                           'illumina' --
                           'pacbio'   --

      -type t            What type of fastq ('sanger' is the default):
                           'sanger'   -- QV's are PHRED, offset=33 '!', NCBI SRA data.
                           'solexa'   -- QV's are Solexa, early Solexa data.
                           'illumina' -- QV's are PHRED, offset=64 '@', Illumina reads from version 1.3 on.
                         See Cock, et al., 'The Sanger FASTQ file format for sequences with quality scores, and
                         the Solexa/Illumina FASTQ variants', doi:10.1093/nar/gkp1137

      -innie             The paired end reads are 5'-3' <-> 3'-5' (the usual case) (default)

      -outtie            The paired end reads are 3'-5' <-> 5'-3' (for Illumina Mate Pair reads)
                         This switch will reverse-complement every read, transforming outtie-oriented
                         mates into innie-oriented mates.  This trick only works if all reads are the
                         same length.

      -reads A           Single ended reads, in fastq format.
      -mates A           Mated reads, interlaced, in fastq format.
      -mates A,B         Mated reads, in fastq format.

Error correction with PacBioToCA
--------------------------------

Since PacBioToCA uses the AMOS package, which is bundled with SMRT Analysis, ensure you have setup the SMRT Analysis environment::

    source /opt/smrtanalysis/etc/setup.sh

Given a frg file of short reads (``illumina.frg``), and a fastq file of long reads (``pacbio.filtered_subreads.fastq``), the following performs error correction::

    /tools/wgs-7.0/Linux-amd64/bin/pacBioToCA -length 500 -partitions 200 -l ec_pacbio -t 16 -s pacbio.spec \
        -fastq pacbio.filtered_subreads.fastq illumina.frg > run.out 2>&1

Some points to consider:

* In general, the ``-length``, ``-partitions``, and ``-t`` parameters can be used without modification.
* ``-l`` specifies the error corrected output filename prefix. The output of the example above will be ``ec_pacbio.frg``, ``ec_pacbio.fasta``, and ``ec_pacbio.qual``.
* Celera Assembler works best on a cluster, so the SGE-enabled version of ``pacbio.spec`` should be used. This is discussed in more detail below.

Assembly of error corrected reads
---------------------------------

The error corrected reads can then be assembled using any long read assembler. This is what you would call using Celera Assembler::

    /tools/wgs-7.0/Linux-amd64/bin/runCA -p asm -d asm -s asm.spec ec_pacbio.frg > asm.out 2>&1

Some notes:

* The assembled contigs will be at ``asm/9-terminator/asm.ctg.fasta``.
* ``-p asm`` can be used without changes.
* ``-d`` is the output directory.
* Similar to the ``pacbio.spec``, the ``asm.spec`` can be tuned for a cluster environment.

Other notes
-----------

* If using Illumina data for the short reads, look at the `Optional Parameters`_ on the PacBioToCA page for some alternative parameters.
* In the CA 7.0 release, there is a bug that may cause crashes in the PacBioToCA pipeline. The `Known Issues`_ section on the PacBioToCA page has details. I have had good luck compiling from source, which addresses this issue.

.. _Optional Parameters: http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=PacBioToCA#Optional_Parameters
.. _Known Issues: http://sourceforge.net/apps/mediawiki/wgs-assembler/index.php?title=PacBioToCA#Known_Issues

.. _specfiles:

Spec files
----------

There are two spec files used in Celera Assembler hybrid assembly. The first, ``pacbio.spec``, is used in error correction. The second, ``asm.spec``, is used for assembly.

These files contain both algorithm-specific parameters as well as hardware-specific parameters. In general the algorithm parameters can be left alone, but the hardware-specific parameters need to be adjusted to your computing environment. The sample data found on the `PacBioToCA wiki page`_ contains reasonable spec files.

Here is a ``pacbio.spec`` file tuned to our cluster environment::

    stopAfter=overlapper

    # original asm settings
    utgErrorRate = 0.25
    utgErrorLimit = 4.5

    cnsErrorRate = 0.25
    cgwErrorRate = 0.25
    ovlErrorRate = 0.25

    merSize=14

    merylMemory = 128000
    merylThreads = 16

    ovlStoreMemory = 8192

    # grid info
    useGrid = 1
    scriptOnGrid = 1
    frgCorrOnGrid = 1
    ovlCorrOnGrid = 1

    sge = -V -S /bin/sh
    #sge = -V -A assembly
    sgeScript = -pe smp 16
    sgeConsensus = -pe smp 1
    sgeOverlap = -pe smp 4
    sgeFragmentCorrection = -pe smp 2
    sgeOverlapCorrection = -pe smp 1

    #ovlMemory=8GB --hashload 0.7
    ovlHashBits = 25
    ovlThreads = 4
    ovlHashBlockLength = 20000000
    ovlRefBlockSize =  50000000

    # for mer overlapper
    merCompression = 1
    merOverlapperSeedBatchSize = 500000
    merOverlapperExtendBatchSize = 250000

    frgCorrThreads = 2
    frgCorrBatchSize = 100000

    ovlCorrBatchSize = 100000

    # non-Grid settings, if you set useGrid to 0 above these will be used
    merylMemory = 128000
    merylThreads = 4

    ovlStoreMemory = 8192

    ovlConcurrency = 6

    cnsConcurrency = 16

    merOverlapperThreads = 2 
    merOverlapperSeedConcurrency = 6 
    merOverlapperExtendConcurrency = 6

    frgCorrConcurrency = 8
    ovlCorrConcurrency = 16 
    cnsConcurrency = 16

Here is an ``asm.spec`` tuned to our cluster environment::

    cnsErrorRate = 0.10
    ovlErrorRate = 0.10

    overlapper = ovl
    unitigger = bogart
    utgBubblePopping = 1

    merSize = 14

    merylMemory = 128000
    merylThreads = 16

    ovlStoreMemory = 8192

    # grid info
    useGrid = 1
    scriptOnGrid = 1
    frgCorrOnGrid = 1
    ovlCorrOnGrid = 1

    sge = -V -S /bin/sh
    sgeScript = -pe smp 16
    sgeConsensus = -pe smp 1
    sgeOverlap = -pe smp 4
    sgeFragmentCorrection = -pe smp 2
    sgeOverlapCorrection = -pe smp 1

    #ovlMemory=8GB --hashload 0.7
    ovlHashBits = 25
    ovlThreads = 6
    ovlHashBlockLength = 20000000
    ovlRefBlockSize =  5000000

    # for mer overlapper
    merCompression = 1
    merOverlapperSeedBatchSize = 500000
    merOverlapperExtendBatchSize = 250000

    frgCorrThreads = 2 
    frgCorrBatchSize = 100000 

    ovlCorrBatchSize = 100000

    # non-Grid settings, if you set useGrid to 0 above these will be used
    merylMemory = 128000
    merylThreads = 12

    ovlStoreMemory = 8192

    ovlConcurrency = 8

    merOverlapperThreads = 6
    merOverlapperSeedConcurrency = 2
    merOverlapperExtendConcurrency = 2

    frgCorrConcurrency = 8

    ovlCorrConcurrency = 16
    cnsConcurrency = 16

    doToggle=0
    toggleNumInstances = 0
    toggleUnitigLength = 2000

    doOverlapBasedTrimming = 1
    doExtendClearRanges = 2 

