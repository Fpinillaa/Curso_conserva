 🧬 Análisis tipo ADMIXTURE con LEA (sNMF) en R  
### Pipeline completo desde un archivo VCF (.vcf.gz)

Este instructivo describe el flujo de trabajo para realizar un análisis de estructura genética tipo **ADMIXTURE** utilizando el paquete **LEA** en **R**, comenzando desde un archivo VCF comprimido.

---

## 1. 📥 Instalación de R y dependencias en Ubuntu

### 1.1 Actualizar el sistema
```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Instalar R
```bash
sudo apt install r-base r-base-dev -y
```

### 1.3 Instalar dependencias adicionales
```bash
sudo apt install libcurl4-openssl-dev libssl-dev libxml2-dev build-essential -y
```

### 1.4 Verificar instalación
```bash
R --version
```

---

## 2. 📦 Crear carpeta de trabajo
```bash
mkdir -p ~/admix_input
cd ~/admix_input
```

---

## 3. 📁 Copiar archivos de entrada

Ajusta las rutas según tu sistema.

Ejemplo (WSL – Descargas):
```bash
cp /mnt/c/Users/tu_usuario/Downloads/muestras.vcf.gz ~/admix_input/
cp /mnt/c/Users/tu_usuario/Downloads/muestras.pop ~/admix_input/
```

---

## 4. 🧾 Descomprimir el archivo VCF

LEA requiere un archivo `.vcf` no comprimido:

```bash
gunzip -k muestras.vcf.gz
```

---

## 5. 🧠 Abrir R en Ubuntu
```bash
R
```

---

## 6. 🧬 Script completo en R para ejecutar sNMF con LEA

```r
# =============================================
#   Análisis tipo ADMIXTURE con LEA (sNMF)
# =============================================

# ---- Instalar y cargar el paquete LEA ----
if (!requireNamespace("LEA", quietly = TRUE)) {
  install.packages("LEA", repos = "https://cloud.r-project.org")
}
library(LEA)

# ---- Definir rutas ----
vcf_file <- "~/admix_input/muestras.vcf"
geno_file <- "~/admix_input/muestras.geno"
pop_file  <- "~/admix_input/muestras.pop"
out_pdf   <- "~/admix_input/ancestry_sNMF.pdf"

# ---- 1. Convertir VCF a formato GENO ----
cat("Convirtiendo VCF a GENO...\n")
vcf2geno(vcf_file, output.file = geno_file)

# ---- 2. Ejecutar sNMF (ajusta K según tus poblaciones) ----
cat("Ejecutando sNMF...\n")
project <- snmf(geno_file,
                K = 3,
                repetitions = 10,
                project = "new",
                entropy = TRUE)

# ---- 3. Seleccionar mejor repetición (menor cross-entropy) ----
ce <- cross.entropy(project, K = 3)
best_run <- which.min(ce)
Qmat <- Q(project, K = 3, run = best_run)

# ---- 4. Leer archivo .pop ----
pops_df <- read.table(pop_file, header = FALSE, stringsAsFactors = FALSE)
pop_vec <- pops_df$V2

# ---- 5. Verificar coincidencia ----
if (length(pop_vec) != nrow(Qmat)) {
  stop("El número de individuos en .pop y .geno no coincide.")
}

# ---- 6. Calcular promedios de ancestría por población ----
agg <- aggregate(Qmat, by = list(Pop = pop_vec), FUN = mean)
print(agg)

# ---- 7. Definir colores por cluster ----
cluster_cols <- c("#70FAA1", "#FA7E70", "#70ACFA")

# ---- 8. Ordenar individuos por población ----
ord <- order(pop_vec)
Q_plot <- Qmat[ord, ]
pop_plot <- pop_vec[ord]

# ---- 9. Crear gráfico tipo ADMIXTURE ----
pdf(out_pdf, width = 10, height = 5)
barplot(t(Q_plot),
        col    = cluster_cols,
        border = NA,
        space  = 0,
        xlab   = "Individuos",
        ylab   = "Proporción de ancestría",
        main   = "Estructura genética (sNMF – LEA)")

legend("topright", legend = unique(pop_plot), fill = cluster_cols)
dev.off()

cat("✅ Análisis completado. Gráfico guardado en:", out_pdf, "\n")
```

---

## 7. 📊 Resultado del análisis

El archivo generado será:

```text
~/admix_input/ancestry_sNMF.pdf
```

El PDF contiene:

- Proporción de ancestría por individuo  
- Clusters coloreados  
- Individuos ordenados por población  

---

## 8. 📚 Formato requerido de `muestras.pop`

El archivo `muestras.pop` debe tener **dos columnas sin encabezado**:

```text
Individuo   Población
Muestra1    Norte
Muestra2    Norte
Muestra3    Sur
Muestra4    Sur
...
```

---

## 9. 📝 Notas finales

- Ajusta `K` según el número esperado de poblaciones.  
- Modifica los colores en `cluster_cols` según tu preferencia.  
- Este pipeline funciona con cualquier archivo VCF bialélico.  
- Asegúrate de que el `.vcf` y el `.pop` correspondan a los mismos individuos.  

---
