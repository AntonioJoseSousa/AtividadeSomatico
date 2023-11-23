## Atividade Somatico

**Download e instalação  de Ferramentas**

```bash
brew install sratoolkit
```
```bash
pip install parallel-fastq-dump
```
```bash
brew install bwa 
```
```bash
brew install bedtools
```
```bash
brew install vcftools
```
```bash
brew install samtools 
```

**Download do arquivo no NCBI da amostra WP312 (FASTQ)**

```bash

wget -c https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/3.0.0/sratoolkit.3.0.0-ubuntu64.tar.gz

tar -zxvf sratoolkit.3.0.0-ubuntu64.tar.gz

export PATH=$PATH://workspace/somaticoEP2/sratoolkit.3.0.0-ubuntu64/bin/

echo "Aexyo" | sratoolkit.3.0.0-ubuntu64/bin/vdb-config

time parallel-fastq-dump --sra-id SRR8856724 \
--threads 10 \
--outdir ./ \
--split-files \
--gzip
```
## AS Referências do Genoma hg19 (FASTA, VCFs)

**Os arquivos de Referência: Panel of Normal (PoN), Gnomad AF**
```bash
https://console.cloud.google.com/storage/browser/gatk-best-practices/somatic-b37?project=broad-dsde-outreach
```
```bash
wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-WGS-panel-b37.vcf
```
```bash
wget -c https://storage.googleapis.com/gatk-best-practices/somatic-b37/Mutect2-WGS-panel-b37.vcf.idx
```
```bash
wget -c  https://storage.googleapis.com/gatk-best-practices/somatic-b37/af-only-gnomad.raw.sites.vcf
```
```bash
wget -c  https://storage.googleapis.com/gatk-best-practices/somatic-b37/af-only-gnomad.raw.sites.vcf.idx
```

**Arquivo no formato FASTA do genoma humano hg19**

```bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr9.fa.gz
```

**BWA index do arquivo chr9.fa.gz**

```bash
gunzip chr9.fa.gz
```

```bash
bwa index chr9.fa
```

```bash
samtools faidx chr9.fa
```

## BWA-mem para fazer o alinhamento (FASTQ -> BAM)
```bash
NOME=WP312; Biblioteca=Nextera; Plataforma=illumina;
bwa mem -t 10 -M -R "@RG\tID:$NOME\tSM:$NOME\tLB:$Biblioteca\tPL:$Plataforma" chr9.fa SRR8856724_1.fastq.gz SRR8856724_2.fastq.gz > WP312.sam
```
## samtools: fixmate, sort e index

**preenche lacunas**
```bash
time samtools fixmate -@10 WP312.sam WP312.bam
```
**orndena**
```bash
samtools sort -O bam -@6 -m4G -o WP312_sorted.bam WP312.bam
```
**index**
```bash
time samtools index WP312_sorted.bam
```
**abordagem de target sequencing utilizamos o rmdup para remover duplicata de PCR**
```bash
time samtools rmdup WP312_sorted.bam WP312_sorted_rmdup_F4.bam
```
**indexando o arquivo BAM rmdup**
```bash
time samtools index WP312_sorted_rmdup_F4.bam 
```

## Gerando BED do arquivo BAM
```bash
bedtools bamtobed -i WP312_sorted_rmdup_F4.bam > WP312_sorted_rmdup.bed
```
```bash
bedtools merge -i WP312_sorted_rmdup.bed > WP312_sorted_rmdup_merged.bed
```
```bash
bedtools sort -i WP312_sorted_rmdup_merged.bed > WP312_sorted_rmdup_merged_sorted.bed
```

## Cobertura Média

**Pegando arquivo view -F4 do BAM WP312.**
```bash
git clone https://github.com/circulosmeos/gdown.pl.git
./gdown.pl/gdown.pl  https://drive.google.com/file/d/1pTMpZ2eIboPHpiLf22gFIQbXU2Ow26_E/view?usp=drive_link WP312_sorted_rmdup_F4.bam
./gdown.pl/gdown.pl  https://drive.google.com/file/d/10utrBVW-cyoFPt5g95z1gQYQYTfXM4S7/view?usp=drive_link WP312_sorted_rmdup_F4.bam.bai

bedtools coverage -a WP312_sorted_rmdup_merged_sorted.bed \
-b WP312_sorted_rmdup_F4.bam -mean \
> WP312_coverageBed.bed
```
## Filtro por total de reads >=20
```bash
cat WP312_coverageBed.bed | \
awk -F "\t" '{if($4>=20){print}}' \
> WP312_coverageBed20x.bed
```
**Adicionando chr nos VCFs do Gnomad e PoN**
**O arquivo af-only-gnomad.raw.sites.vcf (do bucket somatic-b37) não tem o chrna frente do nome do cromossomo.**
**Precisamos adicionar para não gerar conflito de contigs de referência na hora de executar o GATK.**
```bash
grep "\#" af-only-gnomad.raw.sites.vcf > af-only-gnomad.raw.sites.chr.vcf
grep  "^9" af-only-gnomad.raw.sites.vcf |  awk '{print("chr"$0)}' >> af-only-gnomad.raw.sites.chr.vcf
```
## indexing
  ```bash
  bgzip af-only-gnomad.raw.sites.chr.vcf
  tabix -p vcf af-only-gnomad.raw.sites.chr.vcf.gz
  ```
  
  ```bash
  grep "\#" Mutect2-WGS-panel-b37.vcf > Mutect2-WGS-panel-b37.chr.vcf 
  grep  "^9" Mutect2-WGS-panel-b37.vcf |  awk '{print("chr"$0)}' >> Mutect2-WGS-panel-b37.chr.vcf 
  ```
  
  ```bash
  bgzip Mutect2-WGS-panel-b37.chr.vcf 
  tabix -p vcf Mutect2-WGS-panel-b37.chr.vcf.gz
  ```
## GATK4 instalação (MuTect2 com PoN)


**Download**
```bash
wget -c https://github.com/broadinstitute/gatk/releases/download/4.2.2.0/gatk-4.2.2.0.zip
```

**Descompactar**
```bash
unzip gatk-4.2.2.0.zip 
```

**Gerar arquivo .dict**
```bash
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R chr9.fa -O chr9.dict
```

**Gerar interval_list do chr9**
```bash
./gatk-4.2.2.0/gatk ScatterIntervalsByNs -R chr9.fa -O chr9.interval_list -OT ACGT
```
**Converter Bed para Interval_list**
```bash
./gatk-4.2.2.0/gatk BedToIntervalList -I WP312_coverageBed20x.bed \
-O WP312_coverageBed20x.interval_list -SD chr9.dict
```
## GATK4 - CalculateContamination
```bahs
./gatk-4.2.2.0/gatk GetPileupSummaries \
	-I WP312_sorted_rmdup_F4.bam  \
	-V af-only-gnomad.raw.sites.chr.vcf.gz  \
	-L WP312_coverageBed20x.interval_list \
	-O WP312.table
```
```bash
./gatk-4.2.2.0/gatk CalculateContamination \
-I WP312.table \
-O WP312.contamination.table
```
**GATK4 - MuTect2 Call**
```bash
./gatk-4.2.2.0/gatk Mutect2 \
  -R /workspace/AtividadeSomatico/chr9.fa \
  -I WP312_sorted_rmdup_F4.bam \
  --germline-resource af-only-gnomad.raw.sites.chr.vcf.gz  \
  --panel-of-normals Mutect2-WGS-panel-b37.chr.vcf.gz \
  --disable-sequence-dictionary-validation \
  -L WP312_coverageBed20x.interval_list \
  -O WP312.somatic.pon.vcf.gz
  ```
**GATK4 - MuTect2 FilterMutectCalls**
  ```bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
-R chr9.fa \
-V WP312.somatic.pon.vcf.gz \
--contamination-table WP312.contamination.table \
-O WP312.filtered.pon.vcf.gz
```
```bash
vcf-compare WP312.filtered.pon.vcf.gz /WP312.filtered.chr.vcf.gz
```
