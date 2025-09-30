# Clase 5
Se realizarán análisis exploratorios sobre distintos conjuntos de datos; específicamente, Análisis de Componentes Principales (PCA) y de admixture.

Ese día trabajaremos con archivos `vcf`. Como se mencionó en clases anteriores, el VCF se obtiene luego de: (i) control de calidad y filtrado de las lecturas `fastq`, (ii) alineamiento de las lecturas al genoma de referencia (`fasta`), y (iii) filtrado/depuración del archivo `bam`.

## PCA

El Análisis de Componentes Principales (PCA) es una técnica ampliamente utilizada en estadística y en genómica de poblaciones para reducir la dimensionalidad y resumir la variación genética. Proyecta la variación en un espacio bi- o tridimensional mediante combinaciones lineales de las variables originales (SNPs), desde un punto de vista frecuentista, lo que permite agrupar muestras por similitud genética.

Para realizar el PCA utilizaremos `plink`. Como entrada, `plink` requiere un `vcf` con filtros básicos y con filtrado por [desequilibrio de ligamiento (LD)](https://es.wikipedia.org/wiki/Desequilibrio_de_ligamiento), ya que el PCA asume independencia entre variables; por ello se eliminan SNPs altamente correlacionados.

La salida principal consta de dos archivos:

- `.eigenval`, con los valores propios (varianza explicada por cada componente)
- `.eigenvec`, con las coordenadas de cada muestra en cada componente principal

Por defecto, PLINK --pca calcula los primeros 20 componentes.

### Paso a paso

1. Convertir el VCF a plink
```bash
plink --gzvcf nombre_archivo.vcf.gz --recode --out nombre_archivo_salida --allow-extra-chr
```
> `--recode` -> genera archivos `.ped` y `.map`
> `--allow-extra-chr` -> permite análizar muestras que no corresponden a humanos

2. Identificación de LD
```bash
plink --file nombre_archivo_plink --indep-pairwise 50 5 0.1 --allow-extra-chr
```
> `--indep-pairwise`->  indica la ventana en la cual se buscaran los SNPs (50 SNPs y 5 SNPs de avance, con un umbral de r<sup>2</sup> ≤ 0.1 de LD) que presentan LD

3. Pruning (eliminación de los LD identificados)
```bash
plink --file nombre_archivo_plink --extract nombre_archivo_LD.prune.in --recode vcf --out nombre_archivo_pruned --allow-extra-chr
```
> `--extract` -> Conserva los SNPs independiente
> `--recode vcf` -> Exporta el subconjunto como `vcf`

1. Ejecutar el PCA
```bash
plink --vcf nombre_archivo_pruned.vcf --pca --out nombre_archivo_pruned_PCA --allow-extra-chr
```

### Gráficar PCA

Para graficar el PCA, copia los archivos `.eigenval` y `.eigenvec` a una carpeta desde donde trabajaremos con [`Rstudio`](https://posit.co/download/rstudio-desktop/)

```R

#####################################################################
#                                                                   #
#                      Grafica                                      #
#                             PCA                                   #
#                                                                   #
#####################################################################     

#### Carga librerias ####

library(ggplot2)

#### Carga Datos ####

eigenvec <- read.table("ruta/carpeta/nombre_archivo.eigenvec", header = F)
eigenval <- read.table("ruta/carpeta/nombre_archivo.eigenval", header = F)

eigenvec$V1 <- as.factor(eigenvec$V1)

## Nombre de las columnas
stopifnot(ncol(eigenvec) >= 4)      # Debe tener al menos PC1 y PC2
pc_cols <- paste0("PC", seq_len(ncol(eigenvec) - 2))
colnames(eigenvec) <- c("FID", "IID", pc_cols)
colnames(eigenval) <- "eigenval"

## Varianza explicada
var_exp <- eigenval$eigenval / sum(eigenval$eigenval) * 100

#### Gráfico exploratorio ####

pca_plot1 <- ggplot(eigenvec, aes(x = PC1, y = PC2)) +
  geom_hline(yintercept = 0, linewidth = 0.2, linetype = "dashed") +
  geom_vline(xintercept = 0, linewidth = 0.2, linetype = "dashed") +
  geom_point(size = 2.8, alpha = 0.95) +
  labs(
    title = "PCA: PC1 vs PC2",
    x = sprintf("PC1 (%.1f%%)", var_exp[1]),
    y = sprintf("PC2 (%.1f%%)", var_exp[2])
  ) +
  theme_minimal(base_size = 13) +
  theme(panel.grid.minor = element_blank(),
        axis.title = element_text(face = "bold"))


pca_plot2 <- ggplot(eigenvec, aes(x = PC2, y = PC3)) +
    geom_hline(yintercept = 0, linewidth = 0.2, linetype = "dashed") +
    geom_vline(xintercept = 0, linewidth = 0.2, linetype = "dashed") +
    geom_point(size = 2.8, alpha = 0.95) +
    labs(
      title = "PCA: PC2 vs PC3",
      x = sprintf("PC2 (%.1f%%)", var_exp[2]),
      y = sprintf("PC3 (%.1f%%)", var_exp[3])
    ) +
    theme_minimal(base_size = 13) +
    theme(panel.grid.minor = element_blank(),
          axis.title = element_text(face = "bold"))

## Ver Gráficos ##

print(pca_plot1)
print(pca_plot2)

#### Guardado de los gráficos ####

ggsave("ruta_carpeta/resultados/pca_plot1.svg", plot = pca_plot1, device = "svg", width = 15, height = 13, dpi = 600)

ggsave("ruta_carpeta/resultados/pca_plot2.svg", plot = pca_plot2, device = "svg", width = 15, height = 13, dpi = 600)
```

## Admixture

Admixture es un software que utiliza máxima verosimilitud para estimar las proporciones de ascendencia de cada individuo a partir de un conjunto multilocus de SNPs. Trabaja con un enfoque frecuentista, evaluando qué fracción del genotipo de cada individuo se asigna a distintos grupos ancestrales.

Además, mediante validación cruzada (CV), el análisis ayuda a determinar el número de grupos (K) más plausible, típicamente el que minimiza el error de CV.

### Pasos

1. Convertir el VCF Pruned a formato `.bed`

```bash
plink --vcf nombre_archivo_pruned.vcf --make-bed --out nombre_archivo_salida --allow-extra-chr --double-id
```

2. Filtrado SNPs con > 5% de datos faltantes

```bash
plink --bfile nombre_archivo_salida --geno 0.05 --make-bed --allow-extra-chr --out nombre_archivo_filt
```

> `--geno` -> Excluye variantes con mising data > 5%

3. Ejecutar admixture K=1 ... 10

```bash
for K in {1..10}; do
    echo "   ▶️ Corriendo K=$K..."
    admixture --cv -j<núcleos> nombre_archivo_filt.bed $K | tee nombre_archivo_filt_log${K}.out
done
```
> `--cv` -> Genera el achivo de validación cruzada
> `-j` -> Cantidad de nucleos a utilizar
> `tee` -> muestra la salida en terminal y la guarda en un archivo `.log`


### Graficar ADMX

Para graficar los resultados de **ADMX** puedes usar la aplicación web **[pophelper](http://pophelper.com/)** o hacerlo en **R** (opción preferible si quieres controlar colores, orden de individuos, etiquetas y otros detalles).

#### Opción web (pophelper)

Carga los archivos `.Q` generados por ADMIXTURE (proporciones de ascendencia por individuo). La herramienta permite ajustar colores y exportar las figuras


#### R

Para graficar en **R** basta con el archivo `.Q`.  
El archivo `.P` (frecuencias alélicas por cluster) es **opcional** y no es necesario para el barplot de ancestrías; puede ser útil para otros análisis.

```R
#####################################################################
#                                                                   #
#                      Grafica                                      #
#                             ADMX                                  #
#                                                                   #
#####################################################################  

#### Carga librerias ####

library(ggplot2)
library(dplyr)
library(tidyr)
library(stringr)

#### Carga Datos ####

## Paths
q_dir <- "directorio/datos"
famfile <- file.path(q_dir, "nombre_archivo.fam")
metadata_file <- file.path(q_dir, "metadata.csv")

## Metadata
metadata <- read.csv(metadata_file)

## FAM
fam <- read.table(famfile)
ind_names <- gsub("_.*", "", fam$V2) # limpia los duplicados en los nombres

## Buscar y ordenar archivos .Q
q_files <- list.files(q_dir, pattern = "\\.Q$", full.names = T)
q_files <- sort(q_files)

#### Procesamiento archivos ####

## Función para procesar un archivo .Q
process_qfile <- function(qfile, ind_names, metadata) {
  Q <- read.table(qfile)
  K <- ncol(Q)
  colnames(Q) <- paste0("Pop", 1:K)
  Q$Individuo <- ind_names
  
  # Añadir metadata
  Q <- left_join(Q, metadata, by = "Individuo")
  
  # Pivotar datos
  Q_long <- pivot_longer(Q, cols = matches("^Pop\\d+$"), 
                         names_to = "Poblacion", values_to = "Proporcion")
  Q_long$K <- paste0("K=", K)
  return(Q_long)
}

## Procesar todos los archivos .Q
all_Q_long <- lapply(q_files, process_qfile, ind_names = ind_names, metadata = metadata) %>%
  bind_rows()

##  Ordenar niveles de K (K=1 a K=10)
all_Q_long$K <- factor(all_Q_long$K, levels = paste0("K=", 1:10))

## Orden manual de individuos
orden_manual <- c("agregar_manualmente_muestras_separadas_por_coma")

## Aplicar orden manual
all_Q_long <- all_Q_long %>%
  mutate(Individuo = factor(Individuo, levels = orden_manual))

## Colores fijos para Pop1 - Pop10
pop_colors <- c(
  "Pop1"  = "#2A7BEA", "Pop2"  = "#AB7A71", "Pop3"  = "#93553C",
  "Pop4"  = "#5A2D26", "Pop5"  = "#F2E205", "Pop6"  = "#FFFF33",
  "Pop7"  = "#3C8393", "Pop8"  = "#F781BF", "Pop9"  = "#999999",
  "Pop10" = "#66C2A5"
)

## Agrupar K en bloques de 3 (1-3, 4-6, 7-9, 10)
unique_K <- levels(all_Q_long$K)
K_groups <- split(unique_K, ceiling(seq_along(unique_K)/3))

## Función para graficar un grupo de K en orientación vertical
plot_K_group <- function(K_group, data) {
  data %>%
    filter(K %in% K_group) %>%
    ggplot(aes(x = Individuo, y = Proporcion, fill = Poblacion)) +
    geom_bar(stat = "identity", width = 0.9) +
    facet_wrap(~K, nrow = 3, scales = "free_x") +  # Facetas verticales
    scale_fill_manual(values = pop_colors) +
    labs(title = paste("ADMIXTURE Ancestry Proportions:", paste(K_group, collapse = ", ")),
         x = "Individuos (orden manual)", y = "Proporción de ancestría") +
    theme_minimal(base_size = 14) +
    theme(
      axis.text.x = element_text(angle = 45, hjust = 1, size = 8),
      strip.text = element_text(size = 12, face = "bold"),
      axis.title.x = element_text(margin = margin(t = 10)),
      panel.spacing = unit(1, "lines"),
      legend.position = "right"
    )
}

## Mostrar cada grupo de K en el visualizador
for (i in seq_along(K_groups)) {
  group <- K_groups[[i]]
  print(plot_K_group(group, all_Q_long))
}

#### Guardar cada grupo de K ####
for (i in seq_along(K_groups)) {
  group <- K_groups[[i]]
  plot_name <- paste0("admixture_K_group_", i)  # Nombre base: admixture_K_group_1
  
  ## Generar plot
  plot_obj <- plot_K_group(group, all_Q_long)
  
  ## Guardar como SVG
  ggsave(file.path(output_dir, paste0(plot_name, ".svg")),
         plot = plot_obj, device = "svg", width = 15, height = 13)
  
  ## Guardar como PNG de alta resolución
  ggsave(file.path(output_dir, paste0(plot_name, ".png")),
         plot = plot_obj, device = "png", width = 8, height = 6, dpi = 600)
}
```
