# Guía de análisis reproducible

## 0. Congelar insumos y diseño

El archivo `config/samples.tsv` se deriva de `metadata/samples.template.tsv`. Validar unicidad de `library_id`, rutas existentes, FASTQ single-end, valores `include_initial` y procedencia. La cohorte inicial se selecciona por `include_initial=true` y `group_id` AD02/AD12. AP permanece inventariado con `include_initial=false`. No inferir especie, localidad ni condición biológica del nombre de un archivo.

Definir `individual_id` como planta independiente. Si existen varias bibliotecas o corridas del mismo individuo, conservar una fila por archivo y combinarlas **después** de QC, con read groups diferenciados; el VCF tiene una columna por individuo. Si las tres «réplicas» son plantas distintas, asignar tres `individual_id` distintos. Guardar manifiesto de FASTQ con SHA-256, versión exacta de referencia, protocolo de enzima/adaptadores, comandos, versiones y fecha. Conservar un registro de exclusiones y motivos.

## 1. QC y preprocesamiento (`results/01_qc`)

- FastQC/MultiQC por archivo: calidad por base, longitud, adaptadores, bases ambiguas, número de lecturas, duplicación y sobrerrepresentación.
- Confirmar con el proveedor el diseño 2b-RAD: enzima tipo IIB, longitud esperada del inserto, bases de reconocimiento, adaptadores, índices, demultiplexado y si los FASTQ ya están recortados. Un solo archivo por muestra indica single-end, pero **no** prueba que ya esté limpio.
- Si es necesario, retirar adaptadores/bases técnicas con fastp o cutadapt, sin recortar a ciegas el sitio informativo. Repetir QC y registrar lecturas retenidas. Si el proveedor entregó datos limpios, documentar el procedimiento y evaluar de todos modos.
- Evaluar contaminación y longitud inconsistente. Excluir fallas técnicas con criterios fijados en `config/` antes de ver PCA/FST.

## 2. Referencia y alineamiento (`results/02_reference`, `03_mapping`)

Usar `AtequilanaWebersBlueHAP1_genome.fa` y `AtequilanaWebersBlueHAP1_annotation.gff3` de la **misma versión**. Verificar checksum, nombres y longitudes de contigs, coordenadas GFF3 (1-based), integridad del FASTA y origen/licencia; generar `.fai`, índice del alineador y diccionario de secuencias. Registrar la fracción de referencia cubierta por los fragmentos 2b-RAD.

Alinear cada FASTQ single-end con un alineador de lecturas cortas (p. ej., BWA-MEM2), estableciendo read group (`ID`, `SM=individual_id`, `LB`, `PL`). Ordenar e indexar BAM con samtools. Evaluar tasa de mapeo, mapeo único/MAPQ, profundidad y distribución por locus, discordancias y duplicación. Un exceso de multimapeo puede reflejar secuencias repetidas. Decidir si se marcan duplicados solo tras evaluar que múltiples lecturas idénticas son esperables en fragmentos 2b-RAD de longitud fija; marcar automáticamente y eliminarlos puede descartar señal real. Probar esta decisión y registrar el efecto.

El mapeo entre especies a *A. tequilana* puede favorecer alelos similares a la referencia. Comparar métricas por especie y contemplar un análisis de sensibilidad con umbrales o referencia alternativa si existe.

## 3. SNP calling y filtros (`results/04_variants`)

**Ruta propuesta, sujeta a ploidía y QC:** HaplotypeCaller de GATK en modo GVCF por individuo, genotipado conjunto de AD02/AD12 con GenomicsDBImport y GenotypeGVCFs, selección de SNPs y filtrado documentado. Antes de ejecutar, comprobar ploidía biológica de cada individuo; configurar `--sample-ploidy` apropiadamente. Para muestras de distinta ploidía o resultados ambiguos, validar que el llamador y el análisis posterior representen correctamente los genotipos; no forzar diploidía.

Conservar VCF bruto, VCF genotipado y VCF filtrado con índice y estadísticas. Evaluar distribuciones de DP, GQ, MQ, QUAL, balance alélico y missingness por muestra y locus; fijar umbrales empíricos, registrar recuentos antes/después y probar sensibilidad. Aplicar filtros de SNP bialélico, calidad de genotipo, profundidad mínima y máxima justificadas, missingness, frecuencia de alelo menor apropiada al tamaño muestral y mapeo único. Evitar filtrar variantes por separación AD02/AD12 si luego se estimará esa separación. Retener un conjunto de loci comparables con cobertura suficiente en ambos grupos; examinar pérdida de sitios de restricción. Una variante por fragmento/locus o depuración de LD sirve para PCA y ADMIXTURE; conservar además el VCF sin esa reducción para anotación y otras estadísticas. No trasladar umbrales de WGS sin inspeccionar las características de 2b-RAD.

**Entregables:** `cohort.raw.vcf.gz`, `cohort.filtered.vcf.gz`, `cohort.structure.vcf.gz`, reportes de QC y tabla de filtros/versiones. Estos nombres son contratos propuestos, no archivos existentes.

## 4. PCA, FST y estructura (`results/05_population`)

Usar exactamente el mismo manifiesto de individuos y conjunto filtrado entre análisis; registrar cualquier subcohorte. Para PCA, convertir `cohort.structure.vcf.gz` a PLINK 2, manejar contigs no canónicos de planta explícitamente, comprobar IDs y alelos y calcular PCs con `--pca`. Graficar PC1/PC2 con porcentajes de varianza y etiquetas `group_id`, `species`, `locality`; examinar outliers, missingness y lote.

Para FST, generar listas de `individual_id` por **población biológica definida**, no por archivo. VCFtools `--weir-fst-pop` calcula FST por sitio; reportar también un estimador global y su método, loci utilizables, incertidumbre (p. ej., remuestreo por bloques/loci independientes) y sensibilidad a filtros. AD02 versus AD12 también confunde especie y localidad: la diferencia observada no separa ambos efectos. Con una sola planta por grupo o réplicas técnicas, FST poblacional e incertidumbre no son interpretables; posponer ese resultado.

Para estructura, usar ADMIXTURE con SNPs bialélicos depurados por LD/fragmento, ensayar K desde 1 hasta un máximo justificado por el número de **individuos independientes**, varias semillas y validación cruzada. Guardar errores CV, matrices Q/P, semillas y figuras. Si hay pocos individuos o solo dos grupos definidos, tratar la gráfica como exploratoria y no llamar «poblaciones ancestrales» a agrupaciones frágiles. No correr un rango de K mayor que el número de individuos utilizables.

## 5. Anotación (`results/06_annotation`)

Construir base SnpEff específica con el FASTA y GFF3 de *A. tequilana* concordantes, verificando gene/transcript/CDS, fase y traducciones cuando haya proteínas de referencia. Anotar el VCF filtrado **completo** (antes de reducir a un SNP por fragmento) y registrar la versión del ensamblaje/base. Producir tabla `CHROM, POS, REF, ALT, gene_id, transcript_id, consequence, impact`, distancias a genes cuando proceda y porcentaje sin anotación. Un SNP intergénico cercano no implica regulación; asignar genes candidatos con ventana explícita y análisis de sensibilidad.

Para loci diferenciados, definir previamente estadístico, filtros, control de múltiples pruebas/criterio de selección y tamaño muestral; con muestra insuficiente limitarse a prioridades exploratorias. GO/KEGG requieren correspondencias verificadas de identificadores de genes a ortólogos/funciones y un universo de fondo de **genes cubiertos y testeables por 2b-RAD**, no todos los genes del genoma. Guardar versión, fuente de anotación, mapas de ortología y parámetros.

## 6. Integración futura (`results/07_integration`)

La llave de unión debe ser `individual_id`/muestra biológica junto con tejido, fecha, extracción, lote y unidad experimental. Conservar tablas originales de NMR y DPPH, ABTS, FRAP, TPC, TFC, taninos y demás ensayos en `data/external/`; normalizar identificadores en tablas derivadas versionadas. **No unir tres réplicas de NMR o ensayos a tres plantas genéticas por posición de fila.** Si NMR representa una mezcla o una sola observación de tres réplicas, registrar esa agregación y analizar al nivel común (grupo) con límites de inferencia. Controlar lote, unidad, normalización, valores faltantes y tamaños de muestra. Primero mostrar concordancia descriptiva entre PCA/genotipos, perfiles NMR y fenotipos; cualquier asociación genotipo-metabolito o genotipo-antioxidante necesita individuos pareados, réplicas independientes y control de estructura poblacional.

## Insumos y decisiones pendientes

| Insumo/decisión | Por qué hace falta |
|---|---|
| FASTQ y manifest de nombres, tamaños, SHA-256, número de lecturas | Validar archivos y reproducir la cohorte |
| Mapa de biblioteca → planta → grupo, especie, localidad, réplicas técnicas/biológicas, AP | Definir unidad estadística y exclusiones |
| Protocolo 2b-RAD del proveedor: enzima, adaptadores, longitud, demultiplexado y recorte | Preprocesar sin perder bases informativas |
| Fuente, versión, licencia y hashes del FASTA/GFF3; coincidencia de contigs | Mapeo y anotación concordantes |
| Ploidía verificada por especie/individuo, o evidencia experimental/bioinformática para establecerla | Configurar llamadas y análisis de genotipos |
| Profundidad, calidad y mapeabilidad observadas | Fijar umbrales de filtros tras QC |
| Número suficiente de individuos independientes por población | FST, estructura e incertidumbre defendibles |
| Versiones de herramientas, entornos y recursos de cómputo | Convertir esta guía en pipeline ejecutable |
| IDs pareados y diseño de NMR/fitoquímica (tejido, lote, réplicas, unidades) | Integración válida sin pseudorreplicación |

## Criterio de cierre

No declarar «pipeline reproducible ejecutable» hasta incorporar un gestor de flujos (Snakemake/Nextflow), archivo de configuración validado, entornos o contenedores fijados, comandos exactos, pruebas con datos pequeños y un reporte de ejecución. Esta guía especifica el orden, las decisiones y los artefactos esperados; no representa resultados de muestras aún no inspeccionadas.
