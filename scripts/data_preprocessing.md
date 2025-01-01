
## Creating directories to work on the project:

    By moving to the directory for this project (in my case 'metagenomics_analysis'):
    # creating directories:
    mkdir -p data/{raw_reads,working_data} results/ scripts/ metadata/
    touch README.md # to add info later

## Downloading raw fastq files:
    
    Though the originial research has 40 total raw fastq files, I am using just 6 (randomly) for now
    For that, First I will need SRA ids of the samples I intend to work with,
    Secondly, I need to download the files using those ids and a tool (described below):
    
### 1.Saving the SRA ids for the samples in a text file:
    
    touch data/sra_ids.txt
    echo -e "ERR2143795\nERR2143765\nERR2143789\nERR2143759\nERR2143785\nERR2143778" > data/sra_ids.txt
    cat data/sra_ids.txt

### 2.Downloading files using wget:

    # downloading fastq files:
    cd data/raw_reads
    xargs -a ../wget_links.txt -I {} wget -c "{}"

### Renaming files for readability:

    for file in *_R1.fastq.gz;do 
        prefix="${file:0:4}"
        mv "$file" "${prefix}_R1.fastq.gz"
    done

    for file in *_R2.fastq.gz;do 
        prefix="${file:0:4}"
        mv "$file" "${prefix}_R2.fastq.gz"
    done

    gunzip *.fastq.gz

## To Remeber:

    # Controls are the samples starting with JC1D and JC4B
    # Unenriched samples are ending with JP21 and JP41
    # Fertilized samples are ending with JP2D and JP4D

## Checking the last read of a control sample:

    tail -n 4 JP4D_R1.fastq

## Quality assesment with fastqc:
    
    fastqc -h
    fastqc *.fastq

    mkdir -p ../../results/initial_fastqc_reports
    mv *.zip ../../results/initial_fastqc_reports
    mv *.html ../../results/initial_fastqc_reports
    
## Trimming low quality reads:

    trimmomatic # just checking the parameters and options

    echo -e ">PrefixPE/1\nTACACTCTTTCCCTACACGACGCTCTTCCGATCT\n>PrefixPE/2\nGTGACTGGAGTTCAGACGTGTGCTCTTCCGATCT" > ../TruSeq3-PE.fa
    # I obtained this file from the original data repository (zenodo) itself

#### Trimming all files using for loop:

    for filename in *_R1.fastq;do   
        r1=$filename
        r2=$(echo "$filename" | sed 's/_R1/_R2/')
        trimmed_r1=$(echo "$r1" | sed 's/^/trimmed_'/)
        trimmed_r2=$(echo "$r2" | sed 's/^/trimmed_'/)
        failed_r1=$(echo "$r1" | sed 's/^/trim_failed_'/)
        failed_r2=$(echo "$r2" | sed 's/^/trim_failed_'/)
    
        trimmomatic PE $r1 $r2 $trimmed_r1 $failed_r1 $trimmed_r2 $failed_r2 SLIDINGWINDOW:4:25 MINLEN:35 ILLUMINACLIP:../TruSeq3-PE.fa:2:40:15
    done
    
    # Adapter sequences (to trim) are saved in data directory in file TrueSeq3-PE.fa
    # Sliding window of 4 is used that removes bases if phred score is below 25
    # Also, any trimmed reads with less than 35 bases left is discarded

#### Moving trimmed files to different directory:
    
    # currently, all trimmed files are inside raw_reads directory, which doesnot make sense.
    mkdir ../working_data/trimmed_files && mv ./trim* $_
    cd ../working_data/trimmed_files && ls

## Rerunning fastqc to check quality of trimmed files:

    for file in trimmed_*.fastq;do
        fastqc $file
    done
    
    # moving fastqc reports to different folder:
    ls trimmed_*_fastqc.*
    mkdir -p ../../../results/post_trimming_fastqc_reports && mv trimmed_*_fastqc.* $_
    
    # I am satisifed with the fastqc reports after trimming, just moving on:
    
## Metagenomic assembly:

    mkdir ../assembled_files/

### for loop for assembly

    for r1 in trimmed_*R1.fastq;do
        r2=$(echo $r1 | sed 's/_R1/_R2/')
        assembled=$(echo $r1 | sed 's/_R1.fastq/_assembled/')
        echo $assembled
        metaspades.py -1 "$r1" -2 "$r2" -o ../assembled_files/$assembled
    done

    cd ../assembled_files && ls
    
    # out of these many files, the contigs.fasta and scaffolds.fasta are the ones with assembly
    
    # file with extension *.gfa holds information to visualize the assembly using programs like [Bandage](https://github.com/rrwick/Bandage)
    
    ls */*.gfa

#### Renaming scaffolds and saving into different directory:

    for file in */scaffolds.fasta;do
        sample_id=$(echo $file | sed 's/trimmed_//' | sed 's/_assembled\/scaffolds.fasta//')
        echo $sample_id
        echo $file
    done

    mkdir scaffolds && mv *_scaffolds.fasta $_

#### Renaming contigs and saving into different directory:

    for file in */contigs.fasta;do
        sample_id=$(echo $file | sed 's/trimmed_//' | sed 's/_assembled\/_contigs.fasta//')
        echo $sample_id
        echo $file
    done
    
    mkdir contigs && mv *_contigs.fasta $_

## Metagenomic binning:

    # Binning of all samples using a for loop:
    # for that, I need contigs.fasta (from assembly) and trimmed reads:
    
    cd contigs && ls
    mkdir -p ../../binned_files/

    for file in *_contigs.fasta;do
        sample_id=$(echo "$file" | sed 's/_contigs.fasta//')
        contigs="$file"
        r1="../../trimmed_files/trimmed_${sample_id}_R1.fastq"
        r2="../../trimmed_files/trimmed_${sample_id}_R2.fastq"
        out_file="../../binned_files/${sample_id}"
        mkdir -p $out_file
        run_MaxBin.pl -thread 8 -contig $contigs -reads $r1 -reads2 $r2 -out $out_file >> ../../binned_files/binning.log 2>&1
        echo "Processed sample: ${sample_id}, output saved to ${out_file}" >> ../../binned_files/binning.log   
    done

    ls -lh ../../binned_files/*/*.summary

    ## At this point, I see summary of only 5 samples out of initial 6. 
    ## It is because, for the remaining 1 samples (JC4B), the number of marker genes is less than 1, read more at ../../binned_files/binning.log

### Concatenate multiple fasta bins of each sample to single fasta file:

    cd ../../binned_files/ && ls

    # this step is crucial for downstream analysis in kraken, otherwise bins of the same samples will be treated as multiple samples in downstream analysis.

    for file in */*.001.fasta; do
        id=$(dirname "$file" | cut -d'/' -f1)  # Extract the portion before '/'
        cat "$id"/"$id".*.fasta > "$id".fasta
    done
    
### copying the summary files into results folder:

    mkdir -p ../../../results/summary_binning/
    find . -type f -iname "*.summary" -exec cp {} ../../../results/summary_binning/ \;
    # copied summary of 5 binned samples to results directory
    
### Checking the quality of the assembled genomes

#### only 5 samples (that passed binning) has .fasta output. So:

    passed_folders="JC1D JP2D JP21 JP4D JP41"

    for folder in $passed_folders;do
        mkdir -p "$folder"/checkm_outputs
        checkm taxonomy_wf domain Bacteria -x fasta "$folder" "$folder"/checkm_outputs/
    done

    # here, due to insufficinet memory, I have specified the marker at the domain level, specific only for bacteria; also mentioned that binned files are in fasta format in folder: binned_files
    # the output will be saved in checkm_outputs

## Taxonomic assignment with Kraken2

    #installing kraken db (make sure you have ~100 GB of space for this):
   
    mkdir ../kraken2_db && cd $_
    wget -c https://refdb.s3.climb.ac.uk/kraken2-microbial/hash.k2d 
    # -c means continue to allow resuming without requiring to start over, incase download gets intruppted.
    wget https://refdb.s3.climb.ac.uk/kraken2-microbial/opts.k2d
    wget https://refdb.s3.climb.ac.uk/kraken2-microbial/taxo.k2d
    
    # searching against the created db:
    
    mkdir ../tx_files

    for file in *.fasta;do
        id=$(basename $file | sed 's/.fasta//')
        kraken2 --db ../kraken2_db --threads 12 $file \
        --output ../tx_files/"$id".kraken --report \
        ../tx_files/"$id".report --use-names
    done

    ls ../tx_files/*.report | wc -l # gives 6
    ls ../tx_files/*.kraken | wc -l # gives 5

    # As expected, no assignments has been done for sample JC4B.

    less ../tx_files/JC1D.report

## Visualization with Krona:

    cd ../tx_files
    
    for file in *.kraken;do
        input=$(echo "$file".input)
        output=$(echo "$file" | sed "s/kraken/krona.out.html/")
        cut -f2,3 $file> ../krona_files/$input # taking just 2nd and 3rd columns of kraken file
        ktImportTaxonomy ../krona_files/$input -o ../krona_files/"$output" # converting to taxonomy visulazation file
    done

## Creating final biom file:
    
    # renaming vacant report file: JC4B.report
    head JC4B.report # null file
    mv JC4B.report JC4B_null_report
    
    # creating biom file in json format from all valid reports:
    kraken-biom *.report --fmt json -o final.biom

    head final.biom


    