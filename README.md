# chronotrace

Depurador con viaje en el tiempo y perfilador de ejecución para Python. Graba un script o una función
(variables, call stack, tiempos, memoria y excepciones), detecta anomalías y te deja recorrer la
ejecución hacia adelante y hacia atrás desde el navegador. Todo ocurre en tu máquina.

## Instalación

Requiere Python 3.11 o superior. En PowerShell, desde esta carpeta:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -e .
```

En Linux o macOS, activa el entorno con `source .venv/bin/activate`.

Las únicas dependencias son FastAPI y uvicorn, y solo las usa el visualizador.

## Probar los ejemplos

`--view` graba, analiza y abre el visualizador con un solo comando (Ctrl+C para cerrarlo):

```powershell
python -m chronotrace run --view examples/01_shared_state.py
python -m chronotrace run --view examples/02_memory_leak.py
python -m chronotrace run --view examples/03_slow_recursion.py
```

| Ejemplo | Bug | Qué detecta chronotrace |
| --- | --- | --- |
| `01_shared_state.py` | Copia superficial que comparte una lista, y dos hilos sin lock | La mutación compartida con sus tres dueños; la actualización perdida y la lectura obsoleta que la causó |
| `02_memory_leak.py` | Caché cuyas claves nunca se repiten | La fuga, la línea que más retiene y el contenedor que crece (`procesador.cache 0→60`); el pico de 3 MB; la excepción capturada |
| `03_slow_recursion.py` | Caso base incorrecto y recursión sin memoizar | El NaN y su línea de origen; las llamadas repetidas y con qué argumentos |

El caso base de `03` es un error lógico puro: el programa no falla, solo devuelve un número
equivocado. Ningún detector lo marca; se encuentra recorriendo la traza. Ve a una llamada con `j = 0`
e `i > 0` y mira qué devuelve.

## Comandos

```powershell
python -m chronotrace run mi_script.py arg1 arg2     # graba mi_script.ctrace
python -m chronotrace view mi_script.ctrace          # abre el visualizador
python -m chronotrace analyze mi_script.ctrace       # lista anomalías (--json para procesarlas)
python -m chronotrace verify mi_script.ctrace        # comprueba la reconstrucción
```

Opciones de `run` (van **antes** del nombre del script):

| Opción | Efecto |
| --- | --- |
| `-o ARCHIVO` | Archivo de salida |
| `-k N` | Un keyframe cada N pasos (25) |
| `--max-depth N` / `--max-steps N` | Límites de profundidad (100) y de pasos (50 000) |
| `--stop-at-limit` | Detiene el script al llegar a `--max-steps`: úsalo con bucles infinitos |
| `--include PATRÓN` / `--exclude PATRÓN` | Grabar solo, o ignorar, ciertos archivos. Repetibles |
| `--no-memory` | No mide memoria: graba varias veces más rápido |
| `--redact REGEX` | Oculta variables por nombre, por ejemplo `"token\|clave"` |
| `--checksums` | Guarda el CRC de cada estado real para que `verify` lo compare. Lento |
| `--backend` | `auto`, `monitoring` o `settrace` |
| `--view`, `--port N`, `--no-browser` | Control del visualizador |

Si interrumpes con Ctrl+C, se guarda lo grabado hasta ese momento.

### Desde código

```python
import chronotrace

@chronotrace.record                       # guarda calcular.ctrace en cada llamada
def calcular(x): ...

@chronotrace.record(output="trazas/{func}-{n}.ctrace", keyframe_interval=10)
async def descargar(url): ...

traza = chronotrace.load_trace("calcular.ctrace")
replayer = chronotrace.Replayer(traza)
estado = replayer.get_state_at(120)       # variables y call stack en el paso 120
cambios = replayer.get_changes(120)       # diff respecto al paso 119
anomalias = chronotrace.analyze(traza)
```

## El visualizador

| Acción | Tecla |
| --- | --- |
| Paso anterior / siguiente | `←` `→` |
| Saltar la llamada (mismo nivel) | `Mayús` + `←` `→` |
| Anomalía siguiente / anterior | `N` / `P` |
| Inicio / final | `Inicio` / `Fin` |
| Reproducir o pausar | `Espacio` |

Además: arrastra sobre la línea de tiempo; haz clic en un número de línea para ir a su siguiente
ejecución (`Mayús`: anterior); haz clic en una variable para ver todos los valores que tomó; haz clic
en un frame del call stack para inspeccionarlo sin moverte en el tiempo.

**Un detalle importante:** el evento «línea» se registra justo *antes* de ejecutarla. Los cambios que
ves en un paso los produjo la línea marcada con «← produjo estos cambios».

## Cómo funciona

```
script → backend (sys.monitoring | sys.settrace) → Recorder → SafeSerializer → diff
       → DeltaStore (keyframes + deltas) → .ctrace → Replayer → FastAPI → navegador
                                                   ↘ Analyzer
```

- **Serializado seguro** (`serializer.py`). Nunca ejecuta código tuyo: ni `__repr__`, ni `__eq__`, ni
  properties. Todo tiene límites de profundidad, tamaño y número de nodos. Lo que no se puede leer
  queda como nodo opaco en lugar de fallar.
- **Keyframes + deltas** (`delta.py`, `replayer.py`). Cada paso guarda operaciones `[ruta, valor]` o
  `[ruta]`; cada N pasos se guarda el estado completo. `get_state_at(t)` aplica como mucho N−1 deltas
  con copy-on-write, por lo que retroceder cuesta lo mismo que avanzar.
- **Exactitud en todos los frames.** Si una función muta una lista de quien la llamó, el cambio aparece
  también en el frame de quien la llamó y en los demás hilos, en el mismo paso.
- **Medición limpia.** El tiempo y la memoria del propio tracer se descuentan.
- **`.ctrace`** es JSON con gzip: abrir una traza ajena no ejecuta código.

## Límites conocidos

- Grabar es lento: alrededor de 0,15 ms por paso con memoria y 0,04 ms con `--no-memory` en el ejemplo
  3. Usa `--include` y `--max-depth` para acotar programas grandes.
- De cada contenedor se ven los primeros 30 y los últimos 10 elementos, y hasta 5 niveles. Un cambio
  fuera de esa ventana solo se nota si altera la longitud. Los textos y bytes largos se comparan con
  un CRC.
- Se expanden los objetos de clases tuyas. Los de bibliotecas aparecen como opacos.
- El `#id` de un objeto es único dentro de un paso. Entre pasos lejanos, Python puede reutilizar la
  dirección de un objeto ya liberado.
- Con varios hilos, la memoria se atribuye al hilo que emite el siguiente evento.
- Los detectores son heurísticos: pueden fallar por exceso o por defecto.
- Una traza contiene los valores de tus variables. Usa `--redact` antes de compartirla. El servidor
  solo escucha en `127.0.0.1`.
- Los objetos vivos se observan; no se puede reanudar la ejecución desde un paso pasado.

## Pruebas

```powershell
python -m pip install -e ".[dev]"
python -m pytest
python -m mypy
```
