Allora
============

Allora is an Overlap-Layout-Consensus based algorithm that is designed to assemble PacBio long reads. It's best suited to microbial and smaller genomes (<10 MB). 

Allora requires an input fasta file and a params.xml.

params.xml
----------

Some tips: 

#. Set the genomeSize appropriately. For lambda, let's use 50000.
#. maxIterations is typically set at 10, but for faster performance, this number can be reduced, though the assembly will likely have more contigs.

.. _SMRT Pipe Reference Guide: http://pacbiodevnet.com

Additional parameters are described in the `SMRT Pipe Reference Guide`_.

The following is a sample params.xml:: 

    <smrtpipeSettings>
        <moduleStage name="mapping" editable="true">
            <module label="Allora v1" id="Assembly" editableInJob="true">
                <description>Allora ("a long read assembler") is the de novo
                    assembly algorithm. It is based on the open source assembly software package
                    AMOS, along with additional software components tailored to Pacific
                    Biosciences' long reads and error profile. Allora uses a traditional
                    overlap-layout-consensus approach to iteratively assemble raw reads into
                    contigs. It then outputs these contigs as FASTA sequence and cmp.h5
                    files.
                </description>
                <param name="overlapScoreThreshold" label="Overlap permissiveness">
                    <value>700</value>
                    <select>
                        <option value="1300">Least permissive</option>
                        <option value="1000">Less permissive</option>
                        <option value="700">Normal</option>
                        <option value="500">More permissive</option>
                        <option value="300">Most permissive</option>
                    </select>
                </param>
                <param name="genomeSize" label="Expected genome size (bp)">
                    <value>1000000</value>
                    <input type="text"/>
                    <rule type="number" min="1.0" message="Value must be positive"/>
                </param>
                <param name="maxIterations" label="Maximum number of iterations">
                    <value>10</value>
                    <input type="text"/>
                    <rule type="digits" min="1.0" message="Value must be an integer between 1 and 100" max="100.0"/>
                </param>
                <param name="outputAce" label="Write an ACE output file.">
                    <value>true</value>
                    <input type="checkbox"/>
                </param>
                <param name="outputBank" label="Write an AMOS bank directory.">
                    <value>False</value>
                    <input type="checkbox"/>
                </param>
                <param name="autoParameters" hidden="true">
                    <value>True</value>
                </param>
                <param name="detectChimeras">
                    <value>True</value>
                    <input type="checkbox"/>
                </param>
                <param name="detectChimerasOptions" hidden="true">
                    <value> --detector Iterative:threshold=2 </value>
                </param>
            </module>
        </moduleStage>
    </smrtpipeSettings>

job.sh
------

The job.sh will contain the command to run Allora using SMRT Pipe. Change the following to specify the fasta file containing the reads you wish to assemble::

    smrtpipe.py --params=params.xml input.fasta

Note that since there is only a single input, there is no need for an input.xml and the fasta file is directly specified in the command.

Running Allora
--------------

First ensure you have setup the SMRT Analysis environment::

    source /opt/smrtanalysis/etc/setup.sh

Afterwards, now run ``job.sh``::
    
    source job.sh


