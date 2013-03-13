This is work in progress. I try my best to document and test the scripts but if you have problems please contact [me](mailto:etseng@pacificbiosciences.com).

1. <a href="#whatfor">What the cDNA_primer scripts are for</a>
2. <a href="#prereq">Prerequisite for running the scripts</a>
3. <a href="#detectprimer">Identifying primers in subreads or CCS reads</a>
4. <a href="#extra">Additional scripts that may be useful</a>
5. <a href="#primers">Common primer pairs</a>

***

<a name="whatfor"></a>
## What the cDNA_primer scripts are for
These scripts are for identifying potential full-length (FL) subreads or CCS reads using the 5' and 3' primer ligated to the transcripts during the cDNA library preparation step. See [this wiki section](https://github.com/Magdoll/cDNA_primer/wiki/Understanding-PacBio-transcriptome-data#wiki-getFL) for why this works.

<a name="prereq"></a>
## Prerequisite for running the scripts
* Python 2.7.x
* [BioPython](http://biopython.org/wiki/Main_Page)
* [PBBarcode module](http://www.smrtcommunity.com/Share/Code?id=a1q70000000GsyfAAC&strRecordTypeName=Code)
* [HMMER](http://hmmer.janelia.org/software/archive) (must get HMMER2; HMMER3 won't work)

To identify 5'/3' primers you will need the PB Barcode module. [Download](http://www.smrtcommunity.com/Share/Code?id=a1q70000000GsyfAAC&strRecordTypeName=Code)(DevNet registration required) and follow directions. It will also ask that you install an old version of [HMMER](http://hmmer.janelia.org/software/archive) (must get HMMER2; HMMER3 won't work).

<a name="detectprimer"></a>
## Identifying primers in subreads or CCS reads
**NOTE:** If you only plan to work with CCS reads, you don't really need to go through the following steps. CCS reads are, for > 99% of the time, full-length transcripts. If you map them to transcripts or genomes, the primers and polyA tails flanking the transcript will naturally be excluded. Of course, if you insist on having them trimmed or want to do a sanity check, read on.

Once you have everything installed, you can run the primer-identifying python script:
``PacBioBarcodeIDCCS.py <input.fa> <primers.fa> <output>``

_input.fa_ is the input fasta filename. It is usually _filtered_subreads.fasta_ (for filtered subreads) or _filtered_CCS_subreads.fasta_ (for CCS reads).

_primers.fa_ contains the 5'/3' primer pair(s) used in the cDNA library prep

_output_ is the output directory name

You will need to know the 5' and 3' primers used in the cDNA library preparation step. Different cDNA kits (ex: Clontech, Invitrogen, ExactStart) use different primer pairs. Refer to last section for some common primer pairs. The file format for primers.fa should be:

\>F0<br>
*5' sequence here*<br>
\>R0<br>
*3' sequence here (but in reverse complement)*<br>

If you have more than one set of primers you can add more, just name them F1,R1,F2,R2...

**NOTE:** if you have symmetrical 5'/3' primers, there's a chance my script for identifying polyA tail will fail. To prevent that one of the ways you can do is by adding a few As (like 4As or 5As) before the 3' primer sequence. In this way it's more likely that the 3' primer will be identified as the one following the polyA tail.

If you downloaded [cDNA_primer](https://github.com/Magdoll/cDNA_primer), In example/ I have put a test set that you can play with.

<a name="juice"></a>
### Select for full-length transcripts and trimming away primers, polyA tails
The script to do this is ``scripts/barcode_trimmer.py``. Use ``scripts/barcode_trimmer.py --help`` to get the full set of options.

In the simplest case, the following command will output a subset of the subreads (or CCS reads) that have **both 5' and 3' primers seen**. In the process it will also trim away the primers and any polyA tail.

``scripts/barcode_trimmer.py -i <input_filename> -d <PBBarcode output directory> -o <output_filename>``

For example::
``scripts/barcode_trimmer.py -i filtered_subreads.fasta -d output -o filtered_subreads.53seen_trimmed.fa``

To output everything including subreads (or CCS reads) that have none, one, or both primers seen, use option --output-anyway. It will still trim away primers and poly A tails when they are present.

To output all subreads (or CCS reads) with 5' primer seen (but ok to not see 3' primer), use --right-nosee-ok. Conversely, --left-no-seeok will output all subreads that have 3' but not necessarily 5'.

If the subread is a reverse complement of the actual transcript, then a full-length transcript will be of the following: ``reverse complement of 3' -- polyT tails -- transcript -- reverse complement of 5'``. In this case, _barcode_trimmer.py_ will reverse complement the sequence.


### Additional parameters for _barcode_trimmer.py_
--min-seqlen: output only subreads (in addition to other criteria) greater than defined length

--change-seqid: by default, the subread IDs are unchanged during output. If this option is used then it changes the ID to reflect the trimming and potential reverse-complementarity. 


<a name="explain_info"></a>
### Explanation of .primer_info.txt
After you run barcode_trimmer.py you should get this .txt file which is a table where 5seen/polyAseen/3seen is '1' if it is seen, otherwise '0'. Sequence strand can be determined by observing 5' and/or 3' at either the beginning or end of the sequence.

You can use _.primer_info.txt_ to get the accurate number of 5', 3', and 5'&3' primers seen per subreads and ZMW. Use command:<br>
``scripts/count_5seen.py <primer_info_filename> <fasta_filename>``

For example:<br>
``scripts/count_5seen.py filtered_subreads.53seen_trimmed.fa.primer_info.txt filtered_subreads.fasta``



<a name="extra"></a>
## Additional scripts that may be useful
To get all filtered subreads that did not have CCS:
``scripts/grab_nonCCS_subreads.py <subreads_filename> <CCS_filename> <output_filename>``

For example:
``scripts/grab_nonCCS_subreads.py filtered_subreads.fasta filtered_CCS_subreads.fasta filtered_nonCCS_subreads.fasta``

To sort a fasta file by length:
``scripts/sort_fasta_by_len.py <fasta_filename>``
This will output a _.sorted.fasta_ or _.sorted.fa_ file.

<a name="primers"></a>
## Common primer pairs
**NOTE:** Please refer to your own cDNA kit to make sure you're using the right primers!!

**Invitrogen Superscript iii **<br>
\>F1<br>
TCGTCGGGGACAACTTTGTACAAAAAAGTTGG<br>
\>R1<br>
CCCAACTTTCTTGTACAAAGTTGTCCCC<br>

**Clontech**<br>
\>F0<br>
AAGCAGTGGTATCAACGCAGAGTAC<br>
\>R0<br>
GTACTCTGCGTTGATACCACTGCTT<br>

(NOTE: since Clontech primers are identical on both ends, to distinguish orientation, I often add extra 'A's in front of the 3' primer to mock polyA tails)