## SNP Effect
Calculated SNP effect and mutation load using SNPeff. Download SNPeff directly from their website, create a directory for downloading and running SNPeff. Check for latest versions here - https://pcingola.github.io/SnpEff/download/

Only changes that needs to be made is the snpEFF.config file. Create a folder called data inside snpeff folder. Change the data.dir inside config file to wherever snpeff is installed. 

### Building Database

To build a database you will need a reference fasta file, a gff3/gtf annotation file and if possible a CDS and protein file. All information is here - https://pcingola.github.io/SnpEff/se_build_db/ 

To create database you will have to add the following lines to the .config file -

#organism, version x

organism.genome : organism

In my case -

#cornatus genome, version co1.0

co1.0.genome : cornatus

Create a folder inside data and name it by your organism version name (co1.0 for me). Have the reference, annotation, CDS and protein files there. SNpeff doesn't like gff3 files so if you have just gff3 then convert it to gtf using AGAT.

Change reference fasta filename to sequences.fa and annotation filename to genes.gtf. I did not have protein and CDS files so add the appropriate flags.

` java -jar snpEff.jar build -gtf22 -v co1.0 -noCheckCds -noCheckProtein `

### Running SNPeff

` java -Xmx8g -jar snpEff.jar eff -s /scratch/pawsey0149/supadhyaya/snpeff/snpEff/results/${FILENAME%%.vcf}_summary.html co1.0 /scratch/pawsey0149/supadhyaya/ornate_dragon/calls/${FILENAME} > /scratch/pawsey0149/supadhyaya/snpeff/snpEff/results/${FILENAME%%.vcf}.ann.vcf `

It creates a stats.html files, genes.txt file and annotation.vcf file.
