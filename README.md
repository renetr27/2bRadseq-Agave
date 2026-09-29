# 2b-RADseq de Agave

Base reproducible para analizar lecturas **single-end** de 2b-RADseq mediante el genoma de referencia de *Agave tequilana* Weber Blue HAP1. El análisis inicial incluye **AD02 (*A. angustifolia*, Yonora)** y **AD12 (*A. americana*, Mezquital)**. Las muestras AP se conservan en el inventario, si existen, pero se excluyen de la cohorte inicial mediante `include_initial=false`; nunca se eliminan de los datos originales.

> Estado: diseño del análisis. Este repositorio aún no contiene FASTQ, referencia, GFF3 ni resultados validados. Los comandos de [docs/workflow.md](docs/workflow.md) son una guía que requiere completar metadatos, comprobar el protocolo 2b-RAD y fijar parámetros antes de ejecutarse.

## Organización propuesta

```text
.
├── README.md
├── .gitignore
├── config/
│   └── samples.tsv              # inventario real, generado a partir de la plantilla
├── metadata/
│   └── samples.template.tsv     # columnas sin datos supuestos
├── docs/
│   └── workflow.md              # decisiones, pasos, controles y entregables
├── reference_genome/            # FASTA/GFF3 locales; no subir archivos grandes
│   ├── AtequilanaWebersBlueHAP1_genome.fa
│   └── AtequilanaWebersBlueHAP1_annotation.gff3
├── data/
│   ├── raw/                     # FASTQ inmutables, single-end
│   └── external/                # futuro NMR/fitoquímica, datos originales
├── workflow/                    # futuro Snakefile/Nextflow y entornos fijados
├── scripts/                     # validación, resúmenes y gráficos versionados
└── results/                     # salidas reconstruibles por etapa
    ├── 01_qc/
    ├── 02_reference/
    ├── 03_mapping/
    ├── 04_variants/
    ├── 05_population/
    ├── 06_annotation/
    └── 07_integration/
```

Git no conserva directorios vacíos: se crean cuando se incorporen los insumos. `.gitignore` excluye datos, referencia y resultados; conservar código, configuración, metadatos sin identificadores sensibles, versiones y manifiestos de hashes. Si se requiere publicar datos, depositarlos en un repositorio de datos con acceso y DOI adecuados, y registrar sus identificadores aquí.

## Primeros pasos

1. Copiar `metadata/samples.template.tsv` a `config/samples.tsv`; completar **una fila por biblioteca/FASTQ**, sin inventar réplicas. Confirmar cuáles son individuos biológicos independientes y cuáles son réplicas técnicas. Cada `sample_id` debe coincidir exactamente con el nombre usado en el VCF.
2. Depositar FASTQ single-end en `data/raw/` y los dos archivos de referencia en `reference_genome/`. Comprobar que son la misma versión del ensamblaje y que los nombres de contigs del GFF3 coinciden con el FASTA.
3. Completar los datos pendientes descritos en [la guía](docs/workflow.md#insumos-y-decisiones-pendientes); registrar versiones de programas, parámetros, fecha, hashes SHA-256 y número de lecturas por muestra.
4. Ejecutar primero QC, alineamiento y llamadas de prueba en un subconjunto. Revisar profundidad, mapeo, duplicados y filtros antes de procesar la cohorte. Congelar un VCF final y un manifiesto de muestras para todas las comparaciones.

## Objetivos y límites

La secuencia analítica es **QC → mapeo → SNP calling conjunto → filtros → PCA / FST / estructura → anotación con GFF3 → integración posterior con NMR y fitoquímica**. Las comparaciones poblacionales requieren individuos biológicos independientes suficientes por grupo; tres archivos o réplicas técnicas de una misma planta no equivalen a tres individuos. Una comparación AD02 frente a AD12 con pocos individuos puede describir diferencias entre muestras, pero no estimar con solidez FST poblacional ni inferir estructura. La ploidía de **cada muestra** debe verificarse antes de fijar genotipos diploides: el ensamblaje HAP1 no demuestra la ploidía de todas las especies o individuos.

La anotación asigna posiciones y posibles efectos relativos a *A. tequilana*, no demuestra función causal en AD02/AD12. La cobertura de 2b-RAD está restringida a sitios de restricción y puede variar entre especies por pérdida de sitios o sesgo de mapeo; documentar ese sesgo y evitar extrapolar SNPs observados a todo el genoma.

## Referencias técnicas

- [Método 2b-RAD](https://www.nature.com/articles/nmeth.2023)
- [GATK: genotipado conjunto germinal](https://gatk.broadinstitute.org/hc/en-us/articles/360035535932-Germline-short-variant-discovery-SNPs-Indels)
- [PLINK 2: PCA](https://www.cog-genomics.org/plink/2.0/strat)
- [VCFtools: FST de Weir y Cockerham](https://vcftools.github.io/man_latest.html)
- [ADMIXTURE: manual](https://dalexander.github.io/admixture/admixture-manual.pdf)
- [SnpEff: construcción desde GFF3](https://pcingola.github.io/SnpEff/snpeff/build_db_gff_gtf/)
