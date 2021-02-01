### Controlo de qualidade com FastQC


```
docker run -v ~/NGS/dados:/data -it quay.io/biocontainers/fastqc:0.11.8--1 bash
```
```
fastqc <filename>
```
### Pré-processamento de ficheiros FASTQ - Remoção de adaptadores 
Todos os adaptadores utilizados nas amostras obtidas podem ser encontrados no Trimmomatic, na diretoria adapters. 
```
docker run -v ~/NGS/dados:/data -it quay.io/biocontainers/trimmomatic:0.35--6 bash
cd /usr/local/share/trimmomatic-0.35-6/adapters
```

Por exemplo, na amostra obtida em Espanha ERR4395294_1.fastq os adaptadores utilizados foram os da Nextera. Na pasta dos adaptadores do Trimmomatic temos exemplos de sequências de adaptadores do Nextera. Utilizando o awk podemos verificar se estas estão presentes no ficheiro: 
```
awk '/AGATGTGTATAAGAGACAG/ {print}' ./ERR4395294_1.fastq
```

A utilização do Trimmomatic vai ser diferente de acordo com o tipo de reads (paired end ou single end). 

Exemplo de remoção dos adaptadores com reads paired end:

```
trimmomatic PE input_forward.fq.gz input_reverse.fq.gz output_forward_paired.fq.gz output_forward_unpaired.fq.gz output_reverse_paired.fq.gz output_reverse_unpaired.fq.gz ILLUMINACLIP:/usr/local/share/trimmomatic-0.35-6/adapters/TruSeq3-PE.fa:2:30:10 LEADING:3 TRAILING:3 MINLEN:36 AVGQUAL:28
```
Exemplo de remoção dos adaptadores com reads single end:

```
trimmomatic SE -phred33 input.fq.gz output.fq.gz ILLUMINACLIP:/usr/local/share/trimmomatic-0.35-6/adapters/TruSeq3-SE:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36 AVGQUAL:28
```

### Comandos utilizados para a remoção dos adaptadores dos ficheiros obtidos:

ERR4157959
```
trimmomatic PE ERR4157959_1.fastq ERR4157959_2.fastq pt_forward_paired.fastq.gz pt_forward_unpaired.fastq.gz pt_reverse_paired.fastq.gz pt_reverse_unpaired.fastq.gz ILLUMINACLIP:/usr/local/share/trimmomatic-0.35-6/adapters/TrueSeq3-PE.fa:2:30:10 LEADING:3 TRAILING:3 MINLEN:36 AVGQUAL:28
```

ERR4395294
```
trimmomatic PE ERR4395294_1.fastq ERR4395294_2.fastq es_forward_paired.fastq.gz es_forward_unpaired.fastq.gz es_reverse_paired.fastq.gz es_reverse_unpaired.fastq.gz ILLUMINACLIP:/usr/local/share/trimmomatic-0.35-6/adapters/NexteraPE-PE.fa:2:30:10 LEADING:3 TRAILING:3 MINLEN:36 AVGQUAL:28
```

SRR12447360
```
trimmomatic SE -phred33 SRR12447360.fastq SRR12447360.fq.gz ILLUMINACLIP:/usr/local/share/trimmomatic-0.35-6/adapters/TruSeq3-SE:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:36 AVGQUAL:28
```
Após este passo, voltamos a verificar a qualidade com o FASTQC. 

### Mapeamento dos reads de alta qualidade contra o genoma de referência

```
docker run -v ~/NGS/dados:/data -u root -it biocontainers/bwa:v0.7.17-3-deb_cv1 bash
```

Criação de uma indexação com o algoritmo bwa para a sequência de referência:

```
bwa index Sars_cov_2.ASM985889v3.dna.toplevel.fa
```

Criação de um mapa de coordenadas dos reads:
```
bwa aln <ref_file.fa>  <file.fastq.gz> <file.sai>  
```


Apenas devemos usar Reads de alta qualidade, neste caso serão usados os paired end fastq files e o ficheiro SRR12447360 não irá ser utilizado nos passos seguintes, visto que não apresenta uma boa qualidade:

```
bwa aln Sars_cov_2.ASM985889v3.dna.toplevel.fa es_forward_paired.fastq > es_forward_paired.sai
bwa aln Sars_cov_2.ASM985889v3.dna.toplevel.fa es_reverse_paired.fastq > es_reverse_paired.sai
bwa aln Sars_cov_2.ASM985889v3.dna.toplevel.fa pt_reverse_paired.fastq > pt_reverse_paired.sai
bwa aln Sars_cov_2.ASM985889v3.dna.toplevel.fa pt_forward_paired.fastq > pt_forward_paired.sai
```

#### Criação dos ficheiros sam a partir dos reads mapeados:

```
bwa sampe Sars_cov_2.ASM985889v3.dna.toplevel.fa es_forward_paired.sai es_reverse_paired.sai es_forward_paired.fastq es_reverse_paired.fastq > es_aln_pe.sam

bwa sampe Sars_cov_2.ASM985889v3.dna.toplevel.fa pt_forward_paired.sai pt_reverse_paired.sai pt_forward_paired.fastq pt_reverse_paired.fastq > pt_aln_pe.sam
```


Conversão dos ficheiros sam em bam:

```
docker run -v ~/NGS/dados:/data -u root -it biocontainers/samtools:v1.7.0_cv4 bash
samtools view -Sb es_aln_pe.sam > es_aln_pe.bam
samtools view -Sb pe_aln_pe.sam > pe_aln_pe.bam
```

Ordenação dos reads:

```
samtools sort es_aln_pe.bam -o es_aln_pe_SORT.bam
samtools sort pt_aln_pe.bam -o pt_aln_pe_SORT.bam
```


Obtenção de estatísticas:

```
samtools flagstat pt_aln_pe.bam
samtools flagstat es_aln_pe.bam
```

## Identificação de variantes

#### Verificação dos alinhamentos recorrendo ao samtools:
```
samtools tview es_aln_pe.sort.bam Sars_cov_2.ASM985889v3.101.dna.toplevel.fa -p 21:9417961
samtools tview pt_aln_pe.sort.bam Sars_cov_2.ASM985889v3.101.dna.toplevel.fa -p 21:9417961
```

### Identificação dos variantes e extração para um ficheiro vcf utilizando o bcftools:
```
bcftools mpileup -Ou -f Sars_cov_2.ASM985889v3.dna.toplevel.fa es_aln_pe.sort.bam | bcftools call -Ov -vc > es.raw.vcf
bcftools mpileup -Ou -f Sars_cov_2.ASM985889v3.dna.toplevel.fa pt_aln_pe.sort.bam | bcftools call -Ov -vc > pt.raw.vcf
```
