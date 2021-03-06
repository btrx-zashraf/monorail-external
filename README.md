# monorail-external

This is for helping potential users of the Monorail RNA-seq processing pipeline (alignment/quantification) get started running their own data through it.

Caveat emptor: both the Monorail pipeline itself and this repo are a work in process, not the final product.

If you're reading this and decide to use the pipeline that's great, but you are beta testing it.

Please file issues here as you find them.

Monorail is split into 2 parts:

* recount-pump
* recount-unify

`recount` comes from the fact that Monorail is the way that the data in `recount3+` is refreshed.

However, Monorail also creates data for https://github.com/ChristopherWilks/snaptron

## Requirements

* Container platform (Singularity)
* Pre-downloaded (or pre-built) genome-of-choice reference indexes (e.g. HG38 or GRCM38), see next section of the README for more details
* List of SRA accessions to process or locally accessible file paths of runs to process
* Computational resources (memory, CPU, disk space)

You can specify the number of CPUs to use but the amount of memory used will be dictated by how large the STAR reference index is.
For human it's *30 GBs of useable RAM*.  

Multiple CPUs (cores/threads) are used by the following processes run within the pipeline:

* STAR (uses all CPUs given)
* Salmon (upto 8 CPUs given)
* parallel-fastq-dump (upto 4 CPUs given)
* bamcount (upto 4 CPUs given)

Snakemake itself will parallelize the various steps in the pipeline if they can be run indepdendently and are not taking all the CPUs.

The amount of disk space will be run-dependent, but typically varies from 10's of MBs to 100's of MBs per *run accession* (for human/mouse).

## Getting Reference Indexes

You will need to either download or pre-build the reference index files including the STAR, Salmon, the transcriptome, and HISAT2 indexes used in the Monorail pipeline.

Reference indexes + annotations are already built/extracted for human (HG38, Gencode V26) and mouse (GRCM38, Gencode M23).

For human hg38, `cd` into the path you will use for the `$RECOUNT_REF_HOST` path in the `singularity/run_recount_pump.sh` runner script and then run this script from the root of this repo:

`get_human_ref_indexes.sh`

Similarly for mouse GRCM38, do the same as above but run:

`get_mouse_ref_indexes.sh`

For the purpose of building your own reference indexes, the versions of the 3 tools that use them in recount-pump are:

* STAR 2.7.3a
* Salmon 0.12.0
* HISAT2 2.1.0

For the unifier, run the `get_unify_refs.sh` script with either `hg38` or `grcm38` as the one argument.

## Pump (per-sample alignment stage)

You need to have Singularity running, I'll be using singularity 2.6.0 here because it's what we have been running.

Singularity versions 3.x and up will probably work, but I haven't tested them extensively.

We 2 modes of input: 

* downloading a sequence run from SRA
* local FASTQ files

For local runs, both gzipped and uncompressed FASTQs are supported as well as paired/single ended runs.

The example script below assumes the recount-pump Singularity image is already downloaded/converted and is present in the working directory.
e.g. `recount-rs5-1.0.6.simg`

Check the quay.io listing for up-to-date Monorail Docker images (which can be converted into Singularity images):

https://quay.io/repository/benlangmead/recount-rs5?tab=tags

As of 2020-10-20 version `1.0.6` is a stable release.

### Conversion from Docker to Singularity

We store versions of the monorail pipeline as Docker images in quay.io, however, they can easily be converted to Singularity images once downloaded locally:

```singularity pull docker://quay.io/benlangmead/recount-rs5:1.0.6```

will result in a Singularity image file in the current working directory:

`recount-rs5-1.0.6.simg`

NOTE: any host filesystem path mapped into a running container *must not* be a symbolic link, as the symlink will not be able to be followed within the container.

Also, you will need to set the `$RECOUNT_HOST_REF` path in the script to where ever you download/build the relevant reference indexes (see below for more details).

### SRA input

All you need to provide is the run accession of the sequencing run you want to process via monorail:

Example:

`/bin/bash run_recount_pump.sh /path/to/recount-pump-singularity.simg SRR390728 SRP020237 hg38 10 /path/to/references`

The `/path/to/references` is the full path to whereever the appropriate reference getter script put them.
Note: this path should not include the final subdirectory named for the reference version e.g. `hg38` or `grcm38`.

This will startup a container, download the SRR390728 run accession (paired) from the study SRP020237 using upto 10 CPUs/cores.

### Local input

You will need to provide a label/ID for the dataset (in place of "my_local_run") and the path to at least one FASTQ file.

Example:

Download the following two, tiny FASTQ files:

http://snaptron.cs.jhu.edu/data/temp/SRR390728_125_1.fastq.gz

http://snaptron.cs.jhu.edu/data/temp/SRR390728_125_2.fastq.gz

```
/bin/bash run_recount_pump.sh /path/to/recount-pump-singularity.simg SRR390728 local hg38 20 /path/to/references /path/to/SRR390728_125_1.fastq.gz /path/to/SRR390728_125_2.fastq.gz
```

This will startup a container, attempt to hardlink the fastq filepaths into a temp directory, and process them using up to 20 CPUs/cores.

Important: the script assumes that the input fastq files reside on the same filesystem as where the working directory is, this is required for the container to be able to access the files as the script *hardlinks* them for access by the container (the container can't follow symlinks).

The 2nd mates file path is optional as is the gzip compression.
The pipeline uses the `.gz` extension to figure out if gzip compression is being used or not.

### Additional Options

As of 1.0.5 there is some support for altering how the workflow is run with the following environment variables:

* KEEP_BAM=1
* KEEP_FASTQ=1
* NO_SHARED_MEM=1

An example with all three options using the test local example:

```
export KEEP_BAM=1 && export KEEP_FASTQ=1 && export NO_SHARED_MEM=1 && /bin/bash run_recount_pump.sh /path/to/recount-pump-singularity.simg SRR390728 local hg38 20 /path/to/references /path/to/SRR390728_125_1.fastq.gz /path/to/SRR390728_125_2.fastq.gz
```

This will keep the first pass alignment BAM, the original FASTQ files, and will force STAR to be run in NoSharedMemory mode with respect to it's genome index for the first pass alignment.


## Unifier (aggregation over per-sample pump outputs)

https://quay.io/repository/broadsword/recount-unify?tab=tags

`1.0.2` is a stable version as of 2020-10-22

Follow the same process as for recount-pump (above) to convert to singularity.

The unifier aggregates the following cross sample outputs:

* gene sums
* exon sums
* junction split read counts

The Unifier assumes the directory/file layout and naming of default runs of recount-pump, where a single sample's output is stored like this under one main parent directory:

`<parent_directory>/<sampleID>_att0/`

e.g.

`pump_output/sample1_att0/`
`pump_output/sample2_att0/`
`pump_output/sample3_att0/`
....

The `/path/to/pump/output` argument below references the path to the `<parent_directory>` above.  This path *must* be on the same filesystem as the unifier's working directory (`path/to/working/directory` below).  This is because the unifier script will hardlink the pump's output files into the expected directry hierarchy it needs to run.

To run the Unifier:

```
/bin/bash run_recount_unify.sh /path/to/recount-unifier-singularity.simg <reference_version> /path/to/references /path/to/working/directory /path/to/pump/output /path/to/sample_metadata.tsv <number_cores>
```

`/path/to/references` here may be the same path as used in recount-pump, but it must contain an additional directory: `<reference_version>_unify`.

where `reference_version` is either `hg38` or `grcm38`.

`sample_metadata.tsv` *must* have a header line and at least the following first 2 columns in exactly this order (it can have as many additional columns as desired):

```
study_id<TAB>sample_id...
<study_id1>TAB<sample_id1>...
...
```

`<study>` and `<sample_id>` can be anything that is unique within the set.

If you only want to run one of the 2 steps in the unifier (either gene+exon sums OR junction counts), you can skip the other operation:

```export SKIP_JUNCTIONS=1 && /bin/bash run_recount_unify.sh ...```
to run only gene+exon sums

or

```export SKIP_SUMS=1 && /bin/bash run_recount_unify.sh ...```
to run only junction counts

### Unifier outputs

#### Multi study mode (default)

recount3 compatible sums/counts matrix output directories are in the `/path/to/working/directory` under:

* `gene_sums_per_study`
* `exon_sums_per_study`
* `junction_counts_per_study`

The first 2 are run together and then the junctions are aggregated.

The outputs are further organized by:
`study_loworder/study/`

where `study_loworder` is the last 2 characters of the study ID, e.g. if study is ERP001942, then the output for gene sums will be saved under:
`gene_sums_per_study/42/ERP001942`

Additionally, the Unifier outputs Snaptron ready junction databases and indices:

* `junctions.bgz`
* `junctions.bgz.tbi`
* `junctions.sqlite`

`rail_id`s are also created for every sample_id submitted in the `/path/to/sample_metadata.tsv` file and stored in:

`samples.tsv`

Further, the Unifier will generate Lucene metadata indices based on the `samples.tsv` file for Snaptron:

* `samples.fields.tsv`
* `lucene_full_standard`
* `lucene_full_ws`
* `lucene_indexed_numeric_types.tsv`

Taken together, the above junctions block gzipped files & indices along with the Lucene indices is enough for a minimally viable Snaptron instance.

Intermediate and log files for the Unifier run can be found in `run_files`

### [Historical background] Layout of links to recount-pump output for recount-unifier

If compatibility with recount3 gene/exon/junction matrix formats is required, the output of recount-pump needs to be organized in a specific way for the Unifier to properly produce per-study level matrices as in recount3. 

For example, if you find that you're getting blanks instead of actual integers in the `all.exon_bw_count.pasted.gz` file, it's likely a sign that the input directory hierarchy was not laid out correctly.

Assuming your top level directory for input is called `recount_pump_full`, the expected directory hierarchy for each sequencing run/sample is:

`pump_output_full/study_loworder/study/sample_loworder/sample/monorail_assigned_unique_sample_attempt_id/`

e.g.
`pump_output_full/42/ERP001942/25/ERR204925/ERP001942_in3_att0`

where `ERP001942_in3_att0` contains the original output of recount-pump for the `ERR204925` sample.

The format of the `monorail_assigned_unique_sample_attempt_id` is `originalsampleid_in#_att0`.  `in#` should be unique across all samples.

`study_loworder` and `sample_loworder` are *always* the last 2 characters of the study and sample IDs respectively.

Your study and  sample IDs may be very different the SRA example here, but they should still work in this setup.  However, single letter studies/runs probably won't.
