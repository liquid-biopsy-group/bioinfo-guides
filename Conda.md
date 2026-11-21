Cada proyecto usa su propio entorno conda, descrito en un `environment.yml` dentro del repositorio. Así cualquier persona del grupo puede instalar exactamente las mismas herramientas y versiones.

## 1. Instalación (una sola vez)

Usamos **Miniforge**, el instalador comunitario que trae `conda-forge` por defecto. En la página de [Miniforge](https://github.com/conda-forge/miniforge) están las instrucciones de instalación para macOS y Linux.

Después, configura los canales:

```bash
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

Comprueba la configuración:

```bash
conda config --show channels          # conda-forge primero, después bioconda
conda config --show channel_priority  # strict
```

- **`conda-forge`** tiene la mayoría de librerías generales (Python, R, compiladores).
- **`bioconda`** tiene las herramientas bioinformáticas (samtools, bedtools, modkit...).
- **`strict`** hace que cada paquete salga del canal de mayor prioridad. Evita mezclas incompatibles.

## 2. La norma principal: nunca instalar en `(base)`

`(base)` es el entorno de conda en sí. Si se llena de paquetes, acaba rompiéndose y arrastra a todos los demás. **Crea siempre un entorno por proyecto.**

## 3. Crear un entorno

```bash
conda search samtools                          # versiones disponibles
conda create -n mi-proyecto samtools=1.25 --dry-run   # simula: qué instalaría, sin instalar
conda create -n mi-proyecto samtools=1.25             # instala
conda activate mi-proyecto
```

- **Fija siempre la versión** (`samtools=1.25`, no `samtools`). Sin versión, dos personas que instalan en fechas distintas obtienen versiones distintas.
- **Usa `--dry-run` antes de instalar** para ver qué va a cambiar.

## 4. El `environment.yml` del proyecto

La forma recomendada es **escribirlo a mano**, con solo las herramientas que usa el proyecto y sus versiones:

```yaml
name: mi-proyecto
channels:
  - conda-forge
  - bioconda
dependencies:
  - python=3.12
  - samtools=1.25
  - pandas=2.2
```

Conda resuelve el resto de dependencias. Un archivo así funciona igual en macOS y en Linux (servidor o HPC).

### Exportar un entorno existente

Si ya tienes el entorno creado y quieres generar el archivo:

```bash
# Solo los paquetes que instalaste explícitamente (portable entre sistemas)
conda export -n mi-proyecto --from-history > environment.yml

# Todos los paquetes con su versión exacta (solo reproducible en tu mismo sistema)
conda export -n mi-proyecto --no-builds > environment.yml
```

Después, **revisa el archivo**:

- Borra la línea `prefix:` del final, si aparece: es la ruta de tu ordenador.
- Con `--from-history`, añade la versión a los paquetes que no la tengan.
- Con `--no-builds`, ten en cuenta que un entorno exportado en macOS puede no instalarse en Linux, porque incluye paquetes propios de cada sistema.

> En versiones de conda anteriores a `conda export`, el equivalente es `conda env export` con las mismas opciones.

## 5. Recrear el entorno desde el repositorio

```bash
conda env create -f environment.yml
conda activate mi-proyecto
```

Para probar cambios sin tocar tu entorno de trabajo, crea una copia:

```bash
conda create -n mi-proyecto-prueba --clone mi-proyecto
```

## 6. Comandos útiles

| Comando | Para qué |
|---|---|
| `conda env list` | Ver los entornos que tienes |
| `conda list -n mi-proyecto` | Ver los paquetes de un entorno |
| `conda search samtools` | Buscar un paquete y sus versiones |
| `conda search samtools=1.25 --info` | Detalle de una versión: dependencias, canal |
| `conda deactivate` | Salir del entorno activo |
| `conda remove -n mi-proyecto --all` | Borrar un entorno entero |
| `conda clean --tarballs --index-cache --dry-run` | Ver cuánto espacio libera limpiar la caché |
| `conda clean --tarballs --index-cache` | Limpiar la caché |

Para buscar paquetes también puedes usar [anaconda.org](https://anaconda.org).

## 7. Si algo falla

- **"PackagesNotFoundError":** comprueba el nombre con `conda search` y que los canales están configurados (sección 1). Algunas herramientas tienen otro nombre en bioconda; por ejemplo, modkit es `ont-modkit`.
- **El entorno tarda mucho en resolverse:** suele indicar versiones incompatibles entre sí. Prueba a relajar una versión o a crear el entorno con menos paquetes y añadir el resto después.
- **Funciona en tu ordenador pero no en el servidor:** revisa que el `environment.yml` no venga de un `--no-builds` hecho en otro sistema (sección 4).
