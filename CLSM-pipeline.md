# CLSM MAGs workflow

### Trim reads

Statistics of raw metagenome reads using seqkit v. 2.8.1

```sh
for i in *_R1.fastq.gz
do
	sample="${i%*_*.fastq.gz}"
	echo "${sample}"
	seqkit stats -a -b -j 6  "${sample}"_R1.fastq.gz "${sample}"_R2.fastq.gz >"${sample}".seqkit.stats.txt
done
```

Trim raw metagenome reads using kneaddata v. 0.8.0

```sh
for i in *_R1.fastq.gz
do
	sample="${i%*_*.fastq.gz}"
	echo "${sample}"
	kneaddata --input "${sample}"_R1.fastq.gz --input "${sample}"_R2.fastq.gz --output PATH --threads 12 --output-prefix "${sample}" --trimmomatic /software/anaconda3/pkgs/trimmomatic-0.39-1/share/trimmomatic-0.39-1 --trimmomatic-options SLIDINGWINDOW:4:20 --trimmomatic-options MINLEN:50 --serial --sequencer-source ${adapter} --bypass-trf --run-fastqc-start --run-fastqc-end  --reorder
done
```

### Assemble metagenomes

Generate metagenome assemblies from paired trimmed reads using MEGAHIT v. 1.2.9.

```sh
megahit -t 64 -1 "${sample}".clean_R1.fastq -2 "${sample}".clean_R2.fastq -o PATH --out-prefix "${sample}" --presets meta-large --continue
```

### Bin contigs

#### Perform binning

Bin assembly contigs sample on depths of coverage using metawrap v. 1.3.2.

```sh
metawrap binning  -o  PATH/Initial_binning -t 32 --metabat2 --maxbin2 --concoct -a ${contigs_file} "${sample}".clean_R1.fastq "${sample}".clean_R2.fastq
metawrap bin_refinement -o PATH/Bin_refinement -t 32 -A PATH/Initial_binning/metabat2_bins/ -B PATH/Initial_binning/maxbin2_bins/  -C PATH/Initial_binning/concoct_bins/ -c 50 -x 10
```

### Evaluate the quality of bins

Evaluate quality assessments generated using CheckM v. 1.1.3.

```sh
# Information on genome completeness, contamination, strain heterogeneity
checkm lineage_wf PATH/mags/ PATH/mags_checkm/ -x fa -t 8

# Phylogenetic markers found in each bin
checkm tree_qa PATH/mags_checkm/ > PATH/mags_checkm_tree_qa.txt
```

### Dereplicate MAGs

Dereplicate MAGs at 95% ANI using dRep v. 3.4.0.

```sh
dRep dereplicate -g mags_paths.txt -comp 50 -con 10 -sa 0.95 -p 60 PATH/mags_drep/
```

## Annotate MAGs

### Predict MAGs genes

Predict MAG genes with Prodigal v. 2.6.3.

```sh
for filename in *.fa
	do
	mag=`samplename $filename .fa`
	prodigal -i $filename -a ${mag}_prodigal/${mag}.faa -d ${mag}_prodigal/${mag}.fna -f gff -p meta >${mag}_prodigal/${mag}.gff
done
```

### Annotate gene functions

Annotate MAG gene functions with eggNOG-Mapper v. 2.1.12.

```sh
for filename in *.faa
	do
	faa=`samplename $filename .faa`
	emapper.py -i ${filename} -o PATH/{$faa} --data_dir /path/to/eggnog-mapper/database /path/to/eggnog-mapper/database/eggnog_proteins.dmnd -m diamond --pident 30 --query_cover 30 --dbmem --cpu 24 --seed_ortholog_evalue 1e-10 --temp_dir PATH
done
```

## Classify MAGs

Run GTDB-Tk v. 2.4.0.

```sh
#Classify
gtdbtk classify --mash_db PATH/tmp_dir --genome_dir /path/to/mags/ --out_dir PATH/ -x fa --cpus 16 --pplacer_cpus 16



### Calculate MAG size

Count number of ACTG nucleotides (excludes Ns) in MAG fasta files

```sh
for mag in *.fa
do
	prefix=`samplename $mag .fa`
	quast.py ${mag} -o PATH/${prefix}.report -t 4
done
```

### virus analysis 

Run genomad v. 1.11.0

```sh
for contigs in *.fa
do
	sample=`samplename $contigs .fa`
	genomad end-to-end --cleanup --threads 8 ${contigs} PATH/genomad/${sample} path_to_genomad_db
done
```

Run virsorter v. 2.2.4

```sh
for contigs in *.fa
do
	sample=`samplename $contigs .fa`
	virsorter  run -i ${contigs} -w PATH/virsorter/${sample} -j 8 -d path_to_virsorter_db --include-groups dsDNAphage,NCLDV,RNA,ssDNA,lavidaviridae --tmpdir ${outputPath} --rm-tmpdir --min-score 0.8 --min-length 5000
done
```