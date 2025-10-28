# Clase 6

## Población
Se estimarán los siguientes parámetros: heterocigosidad esperada, heterocigosidad observada, diversidad nucleotídica y estadístico de Tajima’s D.
En el caso de la heterocigosidad, al disponer de ambos valores (esperada y observada) será posible evaluar si las poblaciones se encuentran en equilibrio de Hardy–Weinberg. Aquellas poblaciones que se desvíen de dicho equilibrio podrían estar siendo afectadas por presiones selectivas, cuellos de botella u otros procesos evolutivos.

Por su parte, el estadístico Tajima’s D permite evaluar la distribución de la variabilidad genética bajo un modelo de neutralidad. Este índice puede tomar valores negativos, positivos o cercanos a cero:

Valores cercanos a 0 indican poblaciones que se ajustan al modelo nulo, evidenciando un equilibrio entre mutación y deriva genética.

Valores positivos reflejan un déficit de alelos raros, lo que podría deberse a una reducción poblacional (cuello de botella) o a selección balanceadora.

Valores negativos representan un exceso de alelos raros, lo que puede estar asociado a una expansión poblacional reciente o a selección direccional.

Finalmente, para comenzar el análisis se creará un ambiente de trabajo en el que se instalarán las herramientas necesarias: `vcftools`, `pixy`, `bcftools`, `tabix` y `plink`.

```bash
conda create -n nombre_ambiente -c bioconda -c conda-forge pixy vcftools tabix plink -y
```

Luego de instalar los programas, utilizaremos un script, es decir, un archivo de texto plano ejecutable que contiene las instrucciones necesarias para realizar uno o varios análisis.
En este caso, el script incluirá los análisis de diversidad genética, excepto el correspondiente a pi, el cual se ejecutará por separado.

```bash
#!/bin/bash

# --------------------------------------------------------------------------
# 📁 1. Definición de rutas y variables
# --------------------------------------------------------------------------

# --- Rutas ---
# VCF con todas las muestras juntas
VCF_MASTER="/ruta/absoluta/del/vcf/con/todas/las/muestras/nombre_archivo_vcf"

# Ruta al archivo de texto con las muestras y las poblaciones
POPULATIONS_FILE="/ruta/absoluta/del/txt/con/todas/las/muestras/nombre_archivo_txt"

# Ruta al directiorio de salida
OUTPUT_DIR="/ruta/absoluta/del/directorio/de/salida"

# --- Parámetros de Análisis ---

# Tamaño ventana para análisis de diversidad
PI_WINDOW_SIZE=20000
# Tamaño de la ventana para Tajima's D
TAJIMA_D_BIN_SIZE=10000

# --- Ambiente Conda ---
source /opt/miniconda3/bin/activate
conda activate nombre_del_ambiente_creado_anteriormente

#--------------------------------------------------------------------------
# Análisis de heterocigosidad, Fis y tajima's D
# --------------------------------------------------------------------------

# --- Creación de directorios de trabajo ---
mkdir -p "${OUTPUT_DIR}"
mkdir -p "${OUTPUT_DIR}/1_vcf_by_pop"
mkdir -p "${OUTPUT_DIR}/2_analysis_results"
mkdir -p "${OUTPUT_DIR}/temp_lists"

ANALYSIS_DIR="${OUTPUT_DIR}/2_analysis_results"

echo "--- Iniciando pipeline: Separación, filtro MAF y análisis de diversidad ---"

# --- PASO 1: SEPARAR EL VCF MAESTRO POR POBLACIÓN ---

# Extrae la lista de poblaciones únicas del archivo de poblaciones
for pop in $(cut -f2 "${POPULATIONS_FILE}" | sort -u); do
    
    echo "▶️  Procesando población: ${pop}"
    
    # --- 1.A: Crear lista de individuos para la población actual ---
    LIST_DIR="${OUTPUT_DIR}/temp_lists"
    POP_SAMPLES_LIST="${LIST_DIR}/${pop}_samples.txt"
    
    echo "   1. Extrayendo lista de muestras para ${pop}..."
    awk -v pop_name="${pop}" '$2 == pop_name {print $1}' "${POPULATIONS_FILE}" > "${POP_SAMPLES_LIST}"

    # --- 1.B: Crear VCF específico para esta población ---
    VCF_POP_DIR="${OUTPUT_DIR}/1_vcf_by_pop"
    VCF_POP_PATH="${VCF_POP_DIR}/${pop}.vcf.gz"

    echo "   2. Creando VCF solo con las muestras de ${pop}..."
    # OJO: los que tengan archivos comprimidos deben cambiar el `--vcf` por `--gzvcf`
    vcftools --vcf "${VCF_MASTER}" \
             --keep "${POP_SAMPLES_LIST}" \
             --recode \
             --recode-INFO-all \
             --stdout | bgzip -c > "${VCF_POP_PATH}"

    # --- PASO 2: VERIFICACIÓN Y ANÁLISIS ---
    if [ -s "$VCF_POP_PATH" ]; then
        echo "   ✅ VCF final para ${pop} creado. Iniciando análisis..."

        tabix -p vcf "${VCF_POP_PATH}"

        echo "      Calculando Fis (endogamia) con VCFtools..."
        vcftools --gzvcf "${VCF_POP_PATH}" --het --out "${ANALYSIS_DIR}/vcftools_fis_${pop}"
        
        echo "      Calculando D de Tajima con VCFtools (ventana de ${TAJIMA_D_BIN_SIZE} pb)..."
        vcftools --gzvcf "${VCF_POP_PATH}" \
                 --TajimaD ${TAJIMA_D_BIN_SIZE} \
                 --out "${ANALYSIS_DIR}/vcftools_tajimaD_${pop}"             
        echo "✅ Análisis completado para la población ${pop}."
    else
        echo "   ❌ ADVERTENCIA: El VCF filtrado para ${pop} no se creó o está vacío. Saltando análisis."
    fi
    
    echo "--------------------------------------------------------------------------"
done

echo "--- ¡Todos los procesos han finalizado! ---"
```

Para calcular la diversidad nucleotídica utilizaremos pixy mediante el siguiente script.
Sin embargo, antes de ejecutar el análisis es necesario modificar los archivos que contienen las poblaciones.
Cada archivo debe incluir una segunda columna, separada por un tabulador (`tab`), que indique la población a la que pertenece cada muestra. Esta estructura es requerida para que pixy pueda ejecutar correctamente el análisis.

La edición de estos archivos se realizará con un editor de texto, y posteriormente se guardarán las modificaciones utilizando el comando `nano`.

```bash
#!/bin/bash

# --------------------------------------------------------------------------
# 📁 1. Definición de rutas y variables
# --------------------------------------------------------------------------

# --- Rutas ---

# Ruta al archivo de texto con las muestras y las poblaciones
POPULATIONS_FILE="/ruta/absoluta/del/vcf/con/separado/por/muestras/nombre_archivo_vcf"

# Ruta .txt con las poblaciones
POPULATIONS="/ruta/absoluta/del/txt/con/muestras/por/poblacion/nombre_archivo.txt"

# Ruta al directiorio de salida
OUTPUT_DIR="/ruta/absoluta/del/directorio/de/salida"

# --- Parámetros de Análisis ---

WINDOW_SIZE=10000
N_CORES=12

# --- Ambiente Conda ---
source /opt/miniconda3/bin/activate
conda activate nombre_del_ambiente_creado_anteriormente

#--------------------------------------------------------------------------
# Análisis de diversidad nucelotidica
# --------------------------------------------------------------------------

for VCF in ${VCF_DIR}/*.vcf.gz; do
    # Extrae nombre base (ejemplo: Poblacion4.vcf.gz → Poblacion4)
    POP=$(basename "$VCF" .vcf.gz)

    echo ">>> Ejecutando pixy para ${POP}"

    # Verifica si existe archivo de muestras correspondiente
    POP_FILE="${POPS_DIR}/${POP}_samples.txt"

    # Crea carpeta de salida específica por población
    mkdir -p "${OUT_DIR}/${POP}"

    # Ejecuta pixy
    pixy --vcf "$VCF" \
         --populations "$POPULATIONS" \
         --window_size "$WINDOW_SIZE" \
         --n_cores "$N_CORES" \
         --stats pi \
         --output_folder "${OUT_DIR}/${POP}" \
         --output_prefix "${POP}"

    echo "✅ Finalizado ${POP}"
done

echo "==== Análisis completado para todas las poblaciones ===="
```

### Graficamos

```R
## Librerias

library(readr)
library(dplyr)
library(tidyr)
library(stringr)
library(purrr)
library(ggplot2)
library(forcats)

# ---- Rutas (EDITAR) ----
het_dir <- ""            # carpeta con *.het
out_dir <- ""      # carpeta de salida
pixy_dir <- ""
dir.create(out_dir, showWarnings = FALSE, recursive = TRUE)

# ---- Leer todos los .het ----
het_files <- list.files(het_dir, pattern = "\\.het$", full.names = TRUE)
stopifnot("No se encontraron archivos .het" = length(het_files) > 0)

# ---- Calcular Ho/He e inferir población desde el nombre del archivo ----
het_indiv <- map_dfr(het_files, function(f){
  pop <- basename(f) |> str_remove("\\.het$")
  read_tsv(f, show_col_types = FALSE) |>
    mutate(
      Ho = 1 - `O(HOM)`/N_SITES,
      He = 1 - `E(HOM)`/N_SITES,
      population = pop
    )
})

# ---- Resumen por población ----
het_pop <- het_indiv %>%
  group_by(population) %>%
  summarise(
    n_ind      = n(),
    mean_Ho    = mean(Ho, na.rm = TRUE),
    sd_Ho      = sd(Ho, na.rm = TRUE),
    se_Ho      = sd_Ho / sqrt(n_ind),
    mean_He    = mean(He, na.rm = TRUE),
    sd_He      = sd(He, na.rm = TRUE),
    se_He      = sd_He / sqrt(n_ind),
    mean_Fis   = mean(F,  na.rm = TRUE),
    sd_Fis     = sd(F,    na.rm = TRUE),
    se_Fis     = sd_Fis / sqrt(n_ind),
    .groups = "drop"
  )

# ---- Gráfico: Ho vs He por población (barras con error) ----
het_long <- het_pop %>%
  select(population, mean_Ho, se_Ho, mean_He, se_He) %>%
  pivot_longer(
    cols = -population,
    names_to = c("metric","stat"),  # metric = mean/se ; stat = Ho/He
    names_sep = "_",
    values_to = "value"
  ) %>%
  pivot_wider(names_from = metric, values_from = value)  # crea columnas mean y se

p_het_bar <- ggplot(het_long, aes(x = fct_reorder(population, mean), y = mean, fill = stat)) +
  geom_col(position = position_dodge(width = 0.8)) +
  geom_errorbar(aes(ymin = mean - se, ymax = mean + se),
                width = 0.2, position = position_dodge(width = 0.8)) +
  labs(x = "Población", y = "Heterocigosidad", fill = "Estadístico",
       title = "Heterocigosidad observada (Ho) y esperada (He) por población") +
  theme_bw(base_size = 12)

ggsave(file.path(out_dir, "het_Ho_He_bar.png"), p_het_bar, width = 9, height = 5, dpi = 300)

# ---- Gráfico: Fis por población (violín + boxplot) ----
p_fis <- het_indiv %>%
  ggplot(aes(x = fct_infreq(population), y = F)) +
  geom_violin(trim = FALSE) +
  geom_boxplot(width = 0.15, outlier.shape = NA) +
  labs(x = "Población", y = "Fis",
       title = "Distribución de Fis por población (vcftools --het)") +
  theme_bw(base_size = 12)

ggsave(file.path(out_dir, "fis_violin.png"), p_fis, width = 9, height = 5, dpi = 300)

# ---- Guardar tabla resumen ----
write_tsv(het_pop, file.path(out_dir, "summary_het_population.tsv"))

message("Listo. Salidas en: ", out_dir)

# =======================
# 2) Diversidad nucleotídica (π) con pixy
# =======================

# Buscar *_pi.txt (p. ej. pixy_P4_pi.txt con columnas 'pop', 'chromosome', 'window_pos_1', 'window_pos_2', 'avg_pi', 'no_sites', ...)
pixy_files <- list.files(pixy_dir, pattern = "_pi\\.txt$", full.names = TRUE)
stopifnot("No se encontraron archivos *_pi.txt en pixy_dir" = length(pixy_files) > 0)

read_pixy <- function(f){
  read_tsv(f, show_col_types = FALSE, progress = FALSE) %>%
    rename(
      population = pop,
      CHROM      = chromosome,
      start      = window_pos_1,
      end        = window_pos_2,
      pi         = avg_pi,
      n_sites    = no_sites
    ) %>%
    mutate(source_file = basename(f))
}

pixy_df <- map_dfr(pixy_files, read_pixy) %>%
  filter(!is.na(pi)) %>%
  mutate(n_sites = ifelse(is.na(n_sites), 0, n_sites))

# Resumen por población (media simple y ponderada por n_sites)
pixy_pop <- pixy_df %>%
  group_by(population) %>%
  summarise(
    n_windows  = n(),
    mean_pi    = mean(pi, na.rm = TRUE),
    sd_pi      = sd(pi, na.rm = TRUE),
    se_pi      = sd_pi / sqrt(n_windows),
    wmean_pi   = if (sum(n_sites, na.rm = TRUE) > 0)
      weighted.mean(pi, w = n_sites, na.rm = TRUE)
    else NA_real_,
    # error estándar aproximado por ventanas (para barras con error)
    wsd_pi     = sd(pi, na.rm = TRUE),
    wse_pi     = wsd_pi / sqrt(n_windows),
    .groups = "drop"
  )

# (Opcional) Resumen por población y cromosoma
pixy_pop_chr <- pixy_df %>%
  group_by(population, CHROM) %>%
  summarise(
    n_windows  = n(),
    mean_pi    = mean(pi, na.rm = TRUE),
    wmean_pi   = if (sum(n_sites, na.rm = TRUE) > 0)
      weighted.mean(pi, w = n_sites, na.rm = TRUE)
    else NA_real_,
    .groups = "drop"
  )

# ---- Gráfico: Distribución de π por población (violín + boxplot) ----
p_pi_violin <- ggplot(pixy_df, aes(x = fct_infreq(population), y = pi)) +
  geom_violin(trim = FALSE) +
  geom_boxplot(width = 0.15, outlier.shape = NA) +
  labs(x = "Población", y = expression(pi),
       title = expression("Distribución de diversidad nucleotídica " * (pi) * " por población (pixy)")) +
  theme_bw(base_size = 12)
ggsave(file.path(out_dir, "pi_violin.png"), p_pi_violin, width = 9, height = 5, dpi = 300)

# ---- Gráfico: Medias de π por población (simple vs ponderada) ----
pixy_means_long <- pixy_pop %>%
  select(population, mean_pi, se_pi, wmean_pi, wse_pi) %>%
  pivot_longer(-population, names_to = "metric", values_to = "value") %>%
  mutate(se = case_when(
    metric == "mean_pi"  ~ se_pi,
    metric == "wmean_pi" ~ wse_pi,
    TRUE ~ NA_real_
  ),
  label = recode(metric,
                 "mean_pi"  = "Media simple",
                 "wmean_pi" = "Media ponderada (n_sites)")) %>%
  filter(!is.na(se))

p_pi_bar <- ggplot(pixy_means_long,
                   aes(x = fct_reorder(population, value), y = value, fill = label)) +
  geom_col(position = position_dodge(width = 0.8)) +
  geom_errorbar(aes(ymin = value - se, ymax = value + se),
                width = 0.2, position = position_dodge(width = 0.8)) +
  labs(x = "Población", y = expression(bar(pi)),
       fill = "Tipo de media",
       title = expression("Media de " * pi * " por población (simple vs ponderada)")) +
  theme_bw(base_size = 12)
ggsave(file.path(out_dir, "pi_means_bar.png"), p_pi_bar, width = 9, height = 5, dpi = 300)

# ---- Guardar tablas resumen ----
write_tsv(het_pop,       file.path(out_dir, "summary_het_population.tsv"))
write_tsv(pixy_pop,      file.path(out_dir, "summary_pixy_population.tsv"))
write_tsv(pixy_pop_chr,  file.path(out_dir, "summary_pixy_population_by_chr.tsv"))

message("Listo. Resúmenes y figuras guardados en: ", out_dir)
```
## Especies

En el caso de Spheniscus, se realizará un árbol de Neighbour-Joining mediante la utilización del siguiente repositorio: https://github.com/sansubs/vcf2pop
.
Para ello, es necesario descargar el repositorio y abrir el archivo .html que contiene la interfaz principal.

En esta página se debe cargar el archivo VCF y seleccionar las siguientes opciones:

1. Genetic Distance
2. Use SNVs for each pair of genomes
3. Neighbour-Joining tree (Unrooted)
4. Newick tree
5. All

Con estas configuraciones se generará un resultado en texto plano, el cual puede cargarse posteriormente en [iTol](https://itol.embl.de/), donde se visualizará el árbol generado.