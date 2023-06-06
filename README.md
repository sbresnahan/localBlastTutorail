# Local BLAST tutorial using a custom sequence database

## Install conda and required packages in bash/terminal

Download and install [miniconda](https://docs.conda.io/en/latest/miniconda.html). Then, open terminal and create a new environment called "BLAST" to install [BLAST+](https://blast.ncbi.nlm.nih.gov/doc/blast-help/downloadblastdata.html#downloadblastdata) and [seqkit](https://bioinf.shenwei.me/seqkit/). Remember to activate your BLAST environment whenever you use these tools.

```
conda create --name BLAST
conda install -n BLAST -c bioconda blast
conda install -n BLAST -c bioconda seqkit
conda activate BLAST
```

## Retrieve sequence accessions from [NCBI nucleotide](https://www.ncbi.nlm.nih.gov/nucleotide/) in your web browser

Use filters to query the NCBI nucleotide database to retrieve desired sequence accessions, for example:

(((ITS1[All Fields] OR 5.8S[All Fields]) OR 28S[All Fields]) OR ITS2[All Fields]) AND ("0"[SLEN] : "10000"[SLEN]) AND fungi[filter]

Would retrieve accessions for all ITS1-ITS2 region sequences in fungi.

Save these as a plain .txt file by clicking "Send to" "File" and choose format "Accession list".

## Retrieved fasta sequences using [reutils](https://cran.r-project.org/web/packages/reutils/index.html) in R

In this loop, we chunk the retrieval of FASTA sequences of the selected accessions from NCBI. We do this because efetch will hang if you try to retrieve > 10000 sequences at once. Save these to a directory called "fasta" within the working directory.

```
setwd("path/to/working/directory")
library(reutils)
accessions <- read.table("accessions.txt",sep="\t",header=F)[,c(1)]
chunks <- split(accessions,cut(seq_along(accessions),5000,labels = FALSE))
for(i in 1:length(chunks)){
  print(i)
  fetch <- efetch(unlist(chunks[i]), db="nuccore", rettype = "fasta", retmode = "text",strand=1)
  write(content(fetch), file = paste(paste("fasta/",i,sep=""),".fasta",sep=""))
  cat("\014")
}
```

## Prepare local blast sequence database in bash/terminal

Combine the chunks into a single fasta file:

```
cd path/to/fasta/directory
find . -maxdepth 1 -type f -name '*.fasta' -print0 | sort -zV | xargs -0 cat > path/to/save/directory/sequences_temp.fasta
```

Remove spaces and clean up sequence headers:

```
cd path/to/save/directory/
awk '/^>/ {printf("\n%s\n",$0);next; } { printf("%s",$0);}  END {printf("\n");}' < sequences_temp.fasta | sed '/^$/d' > sequences_temp2.fasta 
cat sequences_temp.fasta | perl -pe 's/^>gi\|\d+\|.*\|(.*)\|.*/>$1/' | sed '/^>/ s/ .*//' > sequences_temp3.fasta  
```

Remove duplicate sequences using seqkit:
```
seqkit rmdup -s < sequences_temp3.fasta > sequences_temp4.fasta
awk '/^>/ {printf("\n%s\n",$0);next; } { printf("%s",$0);}  END {printf("\n");}' < sequences_temp4.fasta | sed '/^$/d' > sequences_temp5.fasta 
cat sequences_temp5.fasta | perl -pe 's/^>gi\|\d+\|.*\|(.*)\|.*/>$1/' | sed '/^>/ s/ .*//' > sequences.fasta
```

Make local blast database. Change "localbastdb" to your desired database name.

```
makeblastdb -in sequences.fasta -out localbastdb -dbtype nucl -hash_index
```

## Run blast using your local sequence database. See the [BLAST+ documentation](https://www.ncbi.nlm.nih.gov/books/NBK279690/) for details.

```
blastn -db localbastdb -query query_file.fasta -out output_file.txt
```
