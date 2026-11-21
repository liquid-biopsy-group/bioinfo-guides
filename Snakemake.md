Snakemake encadena los pasos de un análisis como **reglas** que convierten unos archivos en otros. Sabe qué falta por hacer, qué se puede ejecutar en paralelo y qué hay que repetir cuando cambia una entrada o un parámetro.

## 1. Instalación

Snakemake se instala en su propio entorno conda (ver [[Conda]]), no en el del proyecto:

```yaml
# environment.yml en la raíz del proyecto
name: snakemake
channels:
  - conda-forge
  - bioconda
dependencies:
  - snakemake=9.27.2
  - graphviz=14.1.2      # para dibujar el grafo del flujo
```

```bash
conda env create -f environment.yml
conda activate snakemake
```

Las herramientas de cada paso (samtools, modkit...) **no** van aquí: cada regla declara su propio entorno (sección 4).

## 2. Dónde va cada cosa

```text
mi-proyecto/
├── config/
│   ├── example.yaml          # se sube: rutas genéricas y parámetros
│   └── local.yaml            # NO se sube: tus rutas reales
├── workflow/
│   ├── Snakefile             # el flujo
│   ├── envs/                 # un entorno conda por herramienta
│   ├── scripts/              # scripts de Python o R que llaman las reglas
│   └── profiles/default/
│       └── config.yaml       # opciones por defecto de la línea de comandos
├── logs/                     # NO se sube
└── results/                  # NO se sube
```

Snakemake encuentra `workflow/Snakefile` y el perfil `workflow/profiles/default/` automáticamente si lanzas los comandos **desde la raíz del proyecto**.

## 3. Una regla

```python
rule sort:
    input:
        "data/{sample}.txt",
    output:
        "results/{sample}.sorted.txt",
    log:
        "logs/sort_{sample}.log",
    shell:
        "sort {input} > {output} 2> {log}"
```

- **`input` / `output`:** los archivos que lee y que produce. Snakemake decide qué ejecutar a partir de ellos.
- **`{sample}` es un comodín.** Su valor se deduce del archivo que se pide: para `results/S1.sorted.txt`, `sample=S1`.
- **`log`:** un log por trabajo, con los mismos comodines que la salida.
- En lugar de `shell` puedes usar `script: "scripts/mi_script.py"`. Dentro del script, `snakemake.input`, `snakemake.output`, `snakemake.params` y `snakemake.wildcards` contienen lo mismo.

## 4. La configuración: `example.yaml` y `local.yaml`

El `Snakefile` lee los valores de ejemplo, que son los que se suben:

```python
configfile: "config/example.yaml"

IN = config["input_dir"]
SAMPLES = config["samples"]


rule all:
    input:
        expand("results/{sample}.sorted.txt", sample=SAMPLES),
```

```yaml
# config/example.yaml
input_dir: /ruta/a/los/datos
samples:
  - S1
  - S2
```

Cada persona copia el ejemplo a `config/local.yaml`, pone sus rutas reales y lanza el flujo con él:

```bash
cp config/example.yaml config/local.yaml
snakemake -n --configfile config/local.yaml
```

Los valores de `local.yaml` **sustituyen** a los de `example.yaml`. Si una clave no aparece en `local.yaml`, se usa la del ejemplo.

- **`rule all` es la primera regla** y lista los resultados finales. Por eso `snakemake` sin argumentos sabe qué tiene que producir.
- **`expand`** genera la lista de archivos: `results/S1.sorted.txt`, `results/S2.sorted.txt`.

### Un entorno conda por regla

```python
rule sort:
    ...
    conda:
        "envs/coreutils.yaml"
```

```yaml
# workflow/envs/coreutils.yaml
channels:
  - conda-forge
dependencies:
  - coreutils=9.5
```

La ruta de `conda:` es relativa al `Snakefile`. Snakemake crea el entorno la primera vez y lo reutiliza.

### El perfil por defecto

```yaml
# workflow/profiles/default/config.yaml
cores: 4
software-deployment-method: conda
```

Con este perfil, `snakemake` equivale a `snakemake --cores 4 --sdm conda`, y nadie tiene que recordar las opciones.

## 5. Ejecutar

**Siempre primero el dry-run:**

```bash
snakemake -n --configfile config/local.yaml
```

Muestra los trabajos que se ejecutarían y por qué (`reason:`), sin ejecutar nada. Si el número de trabajos no es el esperado, para y revisa.

Después:

```bash
snakemake --configfile config/local.yaml
```

| Comando | Para qué |
|---|---|
| `snakemake -n` | Dry-run: qué haría y por qué |
| `snakemake --cores 1 results/S1.sorted.txt` | Pedir un solo archivo, no el flujo entero |
| `snakemake -p` | Mostrar los comandos de terminal que ejecuta |
| `snakemake --forcerun sort` | Repetir una regla y todo lo que depende de ella |
| `snakemake --rerun-incomplete` | Rehacer trabajos que quedaron a medias tras una interrupción |
| `snakemake --keep-going` | Seguir con los trabajos independientes aunque uno falle |
| `snakemake --lint` | Revisar el flujo y sugerir mejoras |
| `snakemake --dag \| dot -Tpng > dag.png` | Grafo de trabajos: una caja por trabajo |
| `snakemake --rulegraph \| dot -Tpng > rulegraph.png` | Grafo de reglas: una caja por regla |

## 6. Qué se vuelve a ejecutar

Snakemake repite un trabajo cuando:

- **Falta su salida.**
- **Ha cambiado una entrada**, o se ha regenerado otra salida de la que depende.
- **Han cambiado sus parámetros, su código o su entorno.**

El dry-run lo explica en la línea `reason:` de cada trabajo. Por ejemplo, si cambias un archivo de entrada, se repite su trabajo y lo que depende de él, pero no el resto de muestras.

## 7. Errores frecuentes

| Error | Causa habitual |
|---|---|
| `MissingInputException` | Falta un archivo de entrada. El mensaje dice cuál (`affected files`). Revisa la ruta en `local.yaml` |
| `Target rules may not contain wildcards` | La primera regla tiene comodines. Pon `rule all` la primera |
| `WildcardError` | Un comodín aparece en `input` o `log` pero no en `output` |
| `AmbiguousRuleException` | Dos reglas pueden producir el mismo archivo. Hay que restringir los comodines o cambiar los nombres de salida |
| `IncompleteFilesException` | Una ejecución anterior se interrumpió. Usa `--rerun-incomplete` |

## 8. Antes de subir el flujo al repositorio

- [ ] `snakemake -n` funciona con `config/example.yaml` cambiando solo las rutas.
- [ ] `snakemake --lint` no da avisos importantes.
- [ ] `config/local.yaml`, `logs/`, `results/` y `.snakemake/` están en el `.gitignore`.
- [ ] El README explica cómo lanzarlo.
