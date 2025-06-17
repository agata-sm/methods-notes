=======================
Nextflow on Bianca
=======================

*17 June 2025*

How to install and run nf-core pipelines on Bianca.

Based on

* https://hackmd.io/@pmitev/nfcore_pipelines_Bianca


Outline
=========

The pipeline has to be installed on Rackham, or on transit (https://docs.uppmax.uu.se/cluster_guides/transit/) because of internet access. This can be done by ``git clone`` or using nf-core tools (likely also taking care of many additional files). In any case, the pipeline should be run on the internet-connected host at least using the test profile, to get all necessary plugins. 

This seems to be best done by own nextflow installation (and possibly also nf-core tools installation). The nf-core pipelines are no longer maintained by Uppmax, the installed versions are from 2023. This also means that the Nextflow module runs an older version, for compatibility.

According to Pavlin's notes, the pipelines can be installed on transit, or at least downloaded there, including building singularity containers. This may help to take care of the plugins (when running test profile). For nextflow and plugins though, ``NXF_HOME`` needs to be set somewhere else than Rackham ``$HOME``.


Instructions
================

The example uses nf-core atacseq dev version.


Pipeline installation via Rackham
-------------------------------------

Nextflow and all pipeline files are installed on Rackham. To make sure all necessary files (plugins!) are transferred, we start from a fresh nextflow installation.

The key is to get fresh ``nextflow`` installation and its ``$NXF_HOME`` directory to be transferred to Bianca.


installation
*****************

1. to install nextflow::

	module load java/OpenJDK_22+36

	export NXF_HOME="/sw/courses/epigenomics/agata/atac-dev/nxf"
	curl -s https://get.nextflow.io | bash


stdout::

      N E X T F L O W
      version 25.04.3 build 5949
      created 02-06-2025 20:56 UTC (22:56 CEST)
      cite doi:10.1038/nbt.3820
      http://nextflow.io


	Nextflow installation completed. Please note:
	- the executable file `nextflow` has been created in the folder: /sw/courses/epigenomics/agata/atac-dev/nxf
	- you may complete the installation by moving it to a directory in your $PATH


check version at the desired ``${NFX_HOME}``::

	${NXF_HOME}/nextflow -v
	nextflow version 25.04.3.5949


2. install nf-core tools via conda environment

Using nf-core tools has advantage over ``git clone`` in that it takes care of the containers and possibly other config issues. It does not, however download plugins - these need to be installed separately (if unknown at the time of installation, these are usually pulled from remote during the test run).

to install nf-core::

	module load conda/latest

	export CONDA_ENVS_PATH="/sw/courses/epigenomics/agata/atac-dev/conda"

	conda create --name nf-core python=3.12 nf-core nextflow

	conda activate nf-core

	#some containers may have been downloaded previously, otherwise it does not matter
	export NXF_SINGULARITY_CACHEDIR="/sw/courses/epigenomics/agata/atac/singularity/"

	nf-core pipelines download atacseq -s singularity 


stdout::

	INFO     Saving 'nf-core/atacseq'
	          Pipeline revision: 'dev'
	          Use containers: 'singularity'
	          Container library: 'quay.io'
	          Using $NXF_SINGULARITY_CACHEDIR': /sw/courses/epigenomics/agata/atac/singularity/'
	          Output directory: 'nf-core-atacseq_dev'
	          Include default institutional configuration: 'False'
	INFO     Processing workflow revision dev, found 38 container images in total.



after all is pulled::

	conda deactivate


**!!Uppmax profile can be found at https://github.com/nf-core/configs/tree/master/conf**

test on Rackham
****************

This should pull the plugins. As of June 2025, the atacseq pipeline requires plugin ``nf-schema@2.3.0``.

To install the desired plugins (making sure $NXF_HOME points to the setup location)::

	${NXF_HOME}/nextflow plugin install nf-schema@2.3.0



The plugin should be present in ``${NXF_HOME}``::


	ll
	total 36
	drwxrwsr-x 3 agata courses  4096 Jun  5 16:38 capsule
	drwxrwsr-x 4 agata courses  4096 Jun  5 16:38 framework
	-rwx--x--x 1 agata agata   17237 Jun  5 16:37 nextflow
	drwxrwsr-x 3 agata courses  4096 Jun  5 16:41 plugins
	drwxrwsr-x 3 agata courses  4096 Jun  5 16:36 tmp

	ls -ah
	.  ..  capsule	framework  nextflow  .nextflow	plugins  tmp

	 ll plugins/
	total 4
	drwxrwsr-x 5 agata courses 4096 Jun  5 16:41 nf-schema-2.3.0


test run is performed using test profile; this probably uses dev config or some other way to cap requested CPUs to 1; not setting resource limits results in an error when submitting jobs (for the atacseq pipeline). The config below worked for the atacseq pipeline test run.


test.config::

	process {
	  resourceLimits = [
	    cpus: 1,
	    memory: 2.GB,
	    time: 1.h
	  ]
	}


params file test_run-params.yml::

	project: uppmax2025-2-292
	outdir: test_run1


test run (on Rackham)::

	module load java/OpenJDK_22+36

	export NXF_HOME="/sw/courses/epigenomics/agata/atac-dev/nxf"
	pipelineDir="/sw/courses/epigenomics/agata/atac-dev/nf-core-atacseq_dev"

	export NXF_SINGULARITY_CACHEDIR="/sw/courses/epigenomics/agata/atac-dev/nf-core-atacseq_dev/singularity-images"


	${NXF_HOME}/nextflow run "${pipelineDir}/dev/main.nf"  \
	-c "${pipelineDir}/uppmax.config" -c test.config \
	-profile singularity,uppmax,test \
	-params-file test_run-params.yml


some warnings of invalid parameters, it worked::

	-[nf-core/atacseq] Pipeline completed successfully-
	Completed at: 05-Jun-2025 10:06:33
	Duration    : 9m 33s
	CPU hours   : 1.0 (7.6% cached)
	Succeeded   : 228
	Cached      : 36


**!!! cannot run test with ``export NXF_OFFLINE='true'``**





Pipeline transfer to Bianca
------------------------------


1. login to transit (session on rackham)

https://uppmax.github.io/UPPMAX-documentation/cluster_guides/transfer_bianca/#transit-server


``ssh agata@transit.uppmax.uu.se``


2. Mount the wharf of your project.

``mount_wharf sens2024608``

uppmax-pwd2FAcode


from rackham (another session), rsync everything except the conda environment::

	rsync -avh nf-core-atacseq_dev agata@transit.uppmax.uu.se:sens2024608/atac-dev

	uppmax-pwd

	rsync -avh ref agata@transit.uppmax.uu.se:sens2024608/atac-dev
	rsync -avh uppmax.config agata@transit.uppmax.uu.se:sens2024608/atac-dev
	rsync -vah nxf agata@transit.uppmax.uu.se:sens2024608/atac-dev

OBS! cp the just set up nextflow home ``${NXF_HOME}`` to Bianca


check if all the expected dirs are present.

test run using **test profile** is not possible on Bianca - the pipeline tries to fetch references from remote




configs
----------------------------------


1. Uppmax config:


2. resource request modifications


simplify config w/o checking max resources::

	cat 7284-atac-nextflow.config
	// limits for Bianca
	process {

	  resourceLimits = [
	    cpus: 16,
	    memory: 256.GB,
	    time: 168.h
	  ]

	    withLabel:process_single {
	        cpus   = { 1 }
	        memory = { 6.GB * task.attempt }
	        time   = { 4.h  * task.attempt }
	    }
	    withLabel:process_low {
	        cpus   = { 2  * task.attempt }
	        memory = { 12.GB * task.attempt}
	        time   = { 4.h   * task.attempt }
	    }
	    withLabel:process_medium {
	    	cpus   = { 6     * task.attempt }
	        memory = { 36.GB * task.attempt}
	        time   = { 16.h  * task.attempt }
	    }
	    withLabel:process_high {
	    	cpus   = { 16 }
	        memory = { 256.GB  }
	        time   = { 24.h  * task.attempt }
	    }
	    withLabel:process_long {
	        time   = { 20.h  * task.attempt}
	    }
	    withLabel:process_high_memory {
	        memory = { 256.GB * task.attempt }
	    }

	}



production run
------------------


run-params.yml::

	project: sens2024608
	outdir: nfcore_atacseq_5vi2025
	input: 7284-samplesheet.csv
	fasta: /proj/nobackup/sens2024608/wharf/agata/agata-sens2024608/atac-dev/ref/Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz
	gtf: /proj/nobackup/sens2024608/wharf/agata/agata-sens2024608/atac-dev/ref/Homo_sapiens.GRCh38.114.gtf.gz
	gff: /proj/nobackup/sens2024608/wharf/agata/agata-sens2024608/atac-dev/ref/Homo_sapiens.GRCh38.114.gff.gz
	blacklist: /proj/nobackup/sens2024608/wharf/agata/agata-sens2024608/atac-dev/ref/ENCFF356LFX.bed
	mito_name: "MT"
	multiqc_title: "7284 nf-core atacseq"
	macs_gsize: 2913022398
	trim_nextseq:
	save_macs_pileup:



run::

	screen

	module load java/OpenJDK_22+36

	export NXF_HOME="/proj/nobackup/sens2024608/wharf/agata/agata-sens2024608/nxf"
	export NXF_SINGULARITY_CACHEDIR="/proj/nobackup/sens2024608/wharf/agata/agata-sens2024608/atac-dev/nf-core-atacseq_dev/singularity-images"
	export NXF_OFFLINE='true'

	pipelineDir="/proj/nobackup/sens2024608/wharf/agata/agata-sens2024608/atac-dev/nf-core-atacseq_dev"

	${NXF_HOME}/nextflow run "${pipelineDir}/dev/main.nf" \
	-c "${pipelineDir}/uppmax.config" -c 7284-atac-nextflow.config \
	-profile singularity \
	-params-file run-params.yml
