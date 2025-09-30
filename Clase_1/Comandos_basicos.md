# Comandos de uso básico en Linux

En este documento encontrarás diferentes **comandos básicos** para Linux, como los presentados en clases y algunos adicionales.

## Estructura de las carpetas

Al trabajar en un sistema Linux te encontrarás con distintos **directorios** (carpetas) y **archivos de texto** con diversas **extensiones**.

Las extensiones indican el **tipo de archivo o lenguaje** y suelen asociarse a formas de interactuar con el sistema o escribir instrucciones que finalmente ejecuta el **hardware** (componentes físicos) del computador.

Algunos ejemplos comunes de lenguajes/formatos son:

- **JavaScript**
- **R**
- **Bash** (intérprete de comandos en GNU/Linux)
- **Python**
- **HTML**
- **Markdown** (para escribir información o documentación)

La estructura de las carpetas puede representarse así:

```bash
- Carpeta_usuario
    |
    |- Carpeta_1
    |- Carpeta_2
        |
        |- Carpeta_3
        |- Carpeta_4
            |
            |- Carpeta_5
            |- Carpeta_6
                |
                |- Carpeta_7
                |- Carpeta_8
                    |
                    |- Archivo_1
                    |- Archivo_2
                    |- Archivo_3
```

Ten en cuenta que este es un **esquema**. En la terminal verás una **línea** con la **ruta** del directorio donde estás:

```bash
nombre_de_usuario:/Carpeta_usuario/Carpeta_2/Carpeta_4/Carpeta_6/Carpeta_8$
```

---

## Comandos

### 1) `cd` — change directory

Comando para moverse de un directorio a otro.

```bash
# Carpeta base
nombre_de_usuario:/Carpeta_usuario/Carpeta_2/Carpeta_4/Carpeta_6/Carpeta_8$

# Retroceder un nivel
cd ..
# -> /Carpeta_usuario/Carpeta_2/Carpeta_4/Carpeta_6

# Retroceder dos niveles
cd ../../
# -> /Carpeta_usuario/Carpeta_2/Carpeta_4

# Entrar a una carpeta concreta
cd Carpeta_5
# -> /Carpeta_usuario/Carpeta_2/Carpeta_4/Carpeta_5
```

---

### 2) `ls` — list

Lista archivos y subdirectorios.

```bash
# Listar archivos de un directorio
ls

# Listar todo, incluidos ocultos (empiezan por .)
ls -a

# Listado detallado (permisos, propietario, tamaño, fecha)
ls -l
```

Salida típica de `ls -l`:

```bash
drwxrwxr-x 4 francisco francisco 4096 ago 17 00:04 Archivo_1
drwxrwxr-x 4 francisco francisco 4096 ago 17 00:04 Archivo_2
-rw-rw-r-- 1 francisco francisco 1024 ago 17 00:04 Archivo_3
```

> La primera columna indica **permisos**; luego vienen **enlaces**, **propietario**, **grupo**, **tamaño**, **fecha** y **nombre**.

---

### 3) `mv` — move

Sirve para **renombrar** y para **mover** archivos o directorios.

```bash
# Renombrar un archivo
mv Archivo_1 Archivo_x
ls
# -> Archivo_2 Archivo_3 Archivo_x

# Mover un archivo a otra carpeta
mv Archivo_x /Carpeta_usuario/Carpeta_2
cd /Carpeta_usuario/Carpeta_2
ls
# -> Archivo_x
```

---

### 4) `mkdir` — make directory

Crea directorios.

```bash
mkdir Carpeta_nueva
ls
# -> Carpeta_nueva
```

---

### 5) `nano` — editor de texto

Crea y edita archivos de texto en la terminal.

```bash
nano prueba.txt
```

Escribe el contenido. Para **guardar y salir**: `Ctrl + X`, luego **Y/S** (confirmar) y **Enter**.

```bash
ls
# -> prueba.txt
```

---

### 6) `cat` — concatenate

Muestra el contenido de un archivo en pantalla (sin abrir editor).

```bash
cat prueba.txt
```

---

### 7) `head` — “encabezado”

Muestra las primeras líneas de un archivo.

```bash
head -n 5 prueba.txt
```

---

### 8) `tail` — “cola”

Muestra las últimas líneas de un archivo.

```bash
tail -n 5 prueba.txt
```

---

### 9) `rm` — remove

Elimina archivos y directorios.

> ⚠️ **Precaución:** lo que borres con `rm` no se recupera fácilmente.

```bash
# Eliminar un archivo
rm prueba.txt

# Eliminar un directorio y su contenido
rm -r Carpeta_nueva
```

---

Con estos comandos podrás **navegar**, **listar**, **crear**, **mover**, **editar** y **eliminar** archivos y carpetas. Existen muchas variantes y opciones; con esta base ya puedes realizar las tareas fundamentales y, con el tiempo, incorporar herramientas que te resulten más prácticas.
