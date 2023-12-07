# Metabarcoding en peces con Qiime2 para el locus 12S

_Martin Holguin Osorio_\
_junio de 2021_\
_Version 3_ 

## Creacion de archivo de metadatos y preparacion de espacio de trabajo
El archivo "metadata.tsv" tiene que seguir un orden  y una organizacion especifica para funcionar corectamente, mas informacion en [tutorial metadata en Qiime2](https://docs.qiime2.org/2020.11/tutorials/metadata/) , se recomienda usar el add-on keemei de google sheets para crear el archivo de "metadata.tsv" de la manera mas rapida y sencilla, mas [info aqui](https://keemei.qiime2.org).

* Tras descargar metadata.tsv usando google sheets:

```
#navego y ubico el archivo de metadata en la direccion de la carpeta donde voy a trabajar(en mi caso /home/martin/eDNA/1 )
cd /home/martin/eDNA/4/12S
#tomo las lecturas de los datos crudos y los pongo dentro de esta nueva carpeta "datos"
mkdir datos
#creo una carpeta donde se ubicaran las salidas de cada proceso (resultados)
mkdir salidas
```

## Importacion de datos 

Luego de ver la naturaleza de estos datos (demultiplexados, con secuenciacion pareada y con los barcodes en las secuencias), defino que tengo que usar otro comando
#para importar los datos, asi que, dejo los nombres de los datos crudos (dejandolos como estan, sin alterar nada) y los ubico en la carpeta "datos"
```
#importo los datos a qiime2
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path datos/ \
  --input-format CasavaOneEightSingleLanePerSampleDirFmt \
  --output-path salidas/secs_multiplexadas.qza
```

## Demultiplexacion 
```
#como estos datos ya estan demultiplexados solo les cambio el nombre
mv salidas/secs_multiplexadas.qza salidas/secs_demultiplexadas.qza
  
#creo resumen de demultiplezacion o mas bien un visualizador de este
qiime demux summarize \
  --i-data salidas/secs_demultiplexadas.qza \
  --o-visualization salidas/secs_demultiplexadas.qzv

#abro visualizador en el navegador
qiime tools view salidas/secs_demultiplexadas.qzv

#en el archivo "untrimmed.qza" quedan todas las secuencias sin asginar
```

## Eliminación de ruido (denoising con DADA2 ) 
```
#hago denoising con los parametros del trabajo de Mathon porque estos parametros rescatan mas diversidad y generan heat_trees mas coloridos (asignan un mayor numero de spp)
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs salidas/secs_demultiplexadas.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 0 \
  --p-trunc-len-r 0 \
  --p-max-ee-f 2 \
  --p-max-ee-r 2 \
  --p-trunc-q 2 \
  --p-chimera-method none \
  --o-table salidas/tabla.qza \
  --o-representative-sequences salidas/secs_representativas.qza \
  --o-denoising-stats salidas/resumen_denoising.qza

#genero visualizacion del archivo "table.qzv", el cual brinda informacion cuantitativa de las secuencias significativas
qiime feature-table summarize \
--i-table salidas/tabla.qza \
--o-visualization salidas/tabla.qzv \
--m-sample-metadata-file metadata.tsv

#resumen de denoising
qiime metadata tabulate \
--m-input-file salidas/resumen_denoising.qza \
--o-visualization salidas/resumen_denoising4.qzv

#secuencias representativas
qiime feature-table tabulate-seqs \
--i-data salidas/secs_representativas.qza \
--o-visualization salidas/secs_representativas.qzv

#visualizo
qiime tools view salidas/tabla.qzv
```


```
# asignacion taxonomica con sklearn
qiime feature-classifier classify-sklearn \
--i-classifi## asignacion taxonomía er ../../BdD/Locales/12S/classifier_12S_v1.qza \
--i-reads salidas/secs_representativas.qza \
--p-confidence 0.70 \
--p-read-orientation same \
--o-classification salidas/taxonomy_12S.qza

#creo visualizador para taxonomia
qiime metadata tabulate \
--m-input-file salidas/taxonomy_12S.qza \
--o-visualization salidas/taxonomy_12S.qzv

#creo un barplot de taxonomia
qiime taxa barplot \
--i-table salidas/tabla.qza \
--i-taxonomy salidas/taxonomy_12S.qza \
--m-metadata-file metadata.tsv \
--o-visualization salidas/taxa_bar_plots.qzv

#visualizo
qiime tools view salidas/taxa_bar_plots.qzv
```

## Filogenia ambiental 

hago filogenia con la base de datos el comando phylogeny align-to-tree-mafft-fasttree hace todo el pipeline para generar la filogenia
*  qiime alignment mafft ...
*  qiime alignment mask ...
*  qiime phylogeny fasttree ...
*  qiime phylogeny midpoint-root ...

Tecnicamente es un pipeline (hace lo mismo en menos lineas) dentro de este pipeline.
```
#hago filogenia ambiental
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences salidas/secs_representativas.qza \
  --output-dir salidas/filogenia_ambiental

#creo visualizador del arbol filogenetico con las spp detectadas en la asignacion taxonomica con sklearn al 90%
qiime empress community-plot \
    --i-tree salidas/filogenia_ambiental/rooted_tree.qza \
    --i-feature-table salidas/tabla.qza \
    --m-sample-metadata-file metadata.tsv \
    --m-feature-metadata-file salidas/taxonomy_12S.qza  \
    --o-visualization salidas/empress-tree.qzv
```





