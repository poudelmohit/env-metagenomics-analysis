
## Creating requirement file to reproduce the analysis:
    
    conda list > requirements.txt # this file has info of all installed packages.
    conda env export > environment.yml # use this file to create exact same conda env later.

## Installing tools in dedicated conda environment: 

    conda env list
    conda create -n mg -y # mg for metagenomics
    conda activate mg

    conda config --add channels defaults
    conda config --add channels bioconda
    conda config --add channels conda-forge

    conda install -y fastqc=0.11.9 trimmomatic=0.39 kraken2=2.1.2 krona=2.8.1 maxbin2=2.2.7 spades=3.15.2 kraken-biom=1.2.0 checkm-genome=1.2.1

    # checking installation:
    which run_MaxBin.pl # for maxbin
    which spades
    which ktImportText # for krona
    which checkm
    which kraken-biom

## Additional Setup:
    
    bash ~/anaconda3/envs/mg/opt/krona/updateTaxonomy.sh 
    wget ftp://ftp.ncbi.nih.gov/pub/taxonomy/taxdump.tar.gz
    tar -xzf taxdump.tar.gz
    mkdir -p .taxonkit/
    cp names.dmp nodes.dmp delnodes.dmp merged.dmp .taxonkit
    rm *dmp readme.txt taxdump.tar.gz gc.prt






    

