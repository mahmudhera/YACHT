# YACHT

YACHT is a mathematically rigorous hypothesis test for the presence or absence of organisms in a metagenomic sample, based on average nucleotide identity.

The associated preprint can be found at:  https://doi.org/10.1101/2023.04.18.537298. Please cite via:
>Koslicki, D., White, S., Ma, C., & Novikov, A. (2023). YACHT: an ANI-based statistical test to detect microbial presence/absence in a metagenomic sample. bioRxiv, 2023-04.


## Installation
### Conda
A conda release will be coming soon. In the meantime, please install manually.

### Manual installation
YACHT requires Python 3.7 or higher. We recommend using a virtual environment (such as conda) 
to install YACHT. To create a virtual environment, run:
```commandline
conda create -n yacht python=3.7
conda activate yacht
```
Then clone the repo:
```commandline
git clone https://github.com/KoslickiLab/YACHT.git
cd YACHT
```
The dependencies can then be installed via:
```bash
conda install -c conda-forge -c bioconda -c anaconda --file requirements.txt
```

## Usage
The workflow for YACHT is as follows: Create sketches of your reference database genomes and of your sample, create a reference dictionary matrix, and then run the YACHT algorithm.

### Creating sketches of your reference database genomes
You will need a reference database in the form of [Sourmash](https://sourmash.readthedocs.io/en/latest/) sketches of a collection of microbial genomes. There are a variety of pre-created databases available at: https://sourmash.readthedocs.io/en/latest/databases.html. Our code uses the "Zipfile collection" format, and we suggest using the [GTDB genomic representatives database](https://farm.cse.ucdavis.edu/~ctbrown/sourmash-db/gtdb-rs207/gtdb-rs207.genomic-reps.dna.k21.zip).

If you want to use a custom database, you will need to create a Sourmash sketch of your FASTA/FASTQ files of your reference database genomes (see [Sourmash documentation](https://sourmash.readthedocs.io/en/latest/) for details). In brief, this can be accomplished via the following:

If you have a single FASTA file with one genome per record:
```bash
sourmash sketch dna -f -p k=31,scaled=1000,abund --singleton <your multi-FASTA file> -o training_database.sig.zip
```

If you have a directory of FASTA files, one per genome:
```bash
# cd into the relevant directory
sourmash sketch dna -f -p k=31,scaled=1000,abund *.fasta -o ../training_database.sig.zip
# cd back to YACHT
```

### Creating sketches of your sample
You will then create a sketch of your sample metagenome, using the same k-mer size and scale factor
```bash
sourmash sketch dna -f -p k=31,scaled=1000,abund -o sample.sig.zip
```

### Creating a reference dictionary matrix
The script `make_training_data_from_sketches.py` collects and transforms the sketched microbial genomes, getting them into a form usable by YACHT. In particular, it removes one of any two organisms that are withing the ANI threshold the user specifies as making two organisms "indistinguishable".
```bash 
python make_training_data_from_sketches.py --ref_file 'gtdb-rs207.genomic-reps.dna.k31.zip' --out_prefix 'gtdb_mut_thresh_0.95' --ani_thresh 0.95
```
The most important parameter of this command is `--ani_thresh`: this is average nucleotide identity (ANI) value below which two organisms are considered distinct. For example, if `--ani_thresh` is set to 0.95, then two organisms with ANI >= 0.95 will be considered indistinguishable. Only the largest of such organisms will be kept in the reference dictionary matrix. The default value of `--ani_thresh` is 0.95. The `--ani_thresh` value chosen here must match the one chosen for the YACHT algorithm (see below).  


### Run the YACHT algorithm
After this, you are ready to perform the hypothesis test for each organism in your reference database. This can be accomplished with something like:
```bash
python run_YACHT.py --ref_matrix 'gtdb_mut_thresh_0.95_ref_matrix_processed.npz' --sample_file 'sample.sig.zip' --ani_thresh 0.95 --significance 0.99 --min_coverage 1 --outfile 'yacht_results.csv'
```
The `--significance` parameter is basically akin to your confidence level: how sure do you want to be that the organism is present? Higher leads to more false negatives, lower leads to more false positives. 
The `--min_coverage` parameter dictates what percentage (value in `[0,1]`) of the distinct k-mers (think: whole genome) must have been sequenced and present in my sample to qualify as that organism as being "present." Setting this to 1 is usually safe, but if you have a very low coverage sample, you may want to lower this value. Setting it higher will lead to more false negatives, setting it lower will lead to more false positives (pretty rapidly).

The output file will be a CSV file; column descriptions can be found [here](docs/column_descriptions.csv). The most important are the following:
* `organism`: The name of the organism
* `pval`: The p-value of the hypothesis test
* 