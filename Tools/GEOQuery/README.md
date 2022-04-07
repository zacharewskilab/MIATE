# Evaluation of GEO dataset adherence to toxicology reporting standards

The [Gene Expression Omnibus (GEO)](https://www.ncbi.nlm.nih.gov/geo/) was queried for experiments using a toxicology study design and adherence to reporting standards evaluated.

1. Use [E-utilities](https://www.ncbi.nlm.nih.gov/books/NBK25497/) to identity datasets with keywords of interest
```console
esearch -db gds -query '(Rattus Norvegicus[ORGANISM] OR Mus Musculus[ORGANISM])\
	AND (dose-response OR toxicology OR exposure OR dose)\
	AND (gse[ETYP])'\
	| efetch -format 'docsum' -mode xml\
	> geo.xml
```


2. Extract titles and FTP links
```console
grep '<FTPLink>' GEO.xml > ftplinks.txt
grep '<Accession>GSE' GEO.xml > accessions.txt
grep -A 1 --no-group-separator '<Accession>GSE' GEO.xml | grep -v '<Accession>GSE' > titles.txt
paste accessions.txt titles.txt ftplinks.txt > merged.geo.txt
```

3. Download the series matrix file containing all the metadata. 
```console
for f in $(cut -f 3 merged.geo.txt)
do
	ftplink=$(echo $f | sed 's=<FTPLink>==' | sed 's=/</FTPLink>==')
	acc=$(echo $ftplink | cut -d '/' -f 7)
	wget ${ftplink}/matrix/${acc}_series_matrix.txt.gz
	gunzip ${acc}_series_matrix.txt.gz
done
```

4. Filter to remove datasets not meeting certain criteria. 
Here we manually filtered to exclude datasets  using _in vitro_ models and non-chemical stressors. A list for reproducibility is available in this repository. 
```console
mkdir filtered_seriesMatrix
while read p
do
	cp $p filtered_seriesMatrix/
done < curated_seriesMatrix.txt
```

5. Extract the characteristic data and reformat in order to find unique terms and values associated with each dataset.
```console
# Extract, combine, and reformat metadata from all datasets.
grep 'Sample_characteristics_ch' GSE*.txt | sed 's/:!/\t/' > GEO_CharacteristicExtracted.txt

# Expand lists of sequential metadata into separate lines (e.g., Strain: C57BL/6, Sex: Male). 
awk 'BEGIN { FS = "\t"}; {for(i = 3; i <= NF; i++) {if(index($3, ":") >=2) print $1"\t"$2"\t"$i; else print $1"\t"$2"\t"$i}}' GEO_CharacteristicExtracted.txt > GEO_CharacteristicExtracted2.txt

# Clean up some formatting
awk -F ':' '{if(NF >= 3) gsub(",", "\t")} {print $0}' GEO_CharacteristicExtracted2.txt > GEO_CharacteristicExtracted3.txt
sed -i 's/"//g' GEO_CharacteristicExtracted3.txt
sed -i 's/:/\t/' GEO_CharacteristicExtracted3.txt
awk 'BEGIN {FS = "\t"} {if(NF > 3) print $0}' GEO_CharacteristicExtracted3.txt > GEO_CharacteristicExtracted4.txt
grep -v 'hybridization date\|donorid\|rat i.d.\|rna extraction date\|Mouse ID\|Heart weight\|subject id\|mouse id\|collected tissue weight (mg)\|body weight (grams)\|barcode\|rin\|wellbarcode\|chiprowid\|chipcolumnid\|protocol\|index\|sample_id\|inventory_sample_name\|bmi\|animal id\|chip antibody\|animal number\|pathology\|body weight gain (%)\|mean coating score the last week before sacrifice' GEO_CharacteristicExtracted4.txt > GEO_CharacteristicExtracted5.txt
cut -f 3,4 GEO_CharacteristicExtracted5.txt | sort -k1 -k2 | uniq > unique_examples.txt
# Terms to exclude which are not of interest for subsequent comparisons.
grep -v '^numi\|^ngene\|^nread\|^weight\|^od260\|^od280\|^percentmito\|^28s\|^animal_id\|^antibod\|^bun\|^bw\|^bioanalyzer\|^library\|^glucose\|^mouse.no\|^mouse_id\|^necrosis\|^Weight\|^sample number\|^scananimal\|^scan\|^sampleid\|^Sample id\|^sample ID\|^sampleID\|^rna\|^ratunique\|^ratio\|^rat id\|^Rat IDs\|^rat identifier\|^OD280\|^OD260\|^mouse number\|^mouse id\|^mouse.id\|^microarray\|^lab sample number\|^individual\|^sample id\|^sampleID' unique_examples.txt > unique_examples2.txt
```

6. Map terms. For reproducibility a file with mapped terms is included in this repository. 
```console
head TermsMapped.txt
```

7. Analyze the data. A template R notebook is included in this repository. 
```console
R -e "rmarkdown::render('GEO_metadata_reporting_standards.Rmd')"
```


