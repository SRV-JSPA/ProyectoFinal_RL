# Semáforos inteligentes en Av. La Reforma con RL multiagente

Proyecto final de CC3104 Aprendizaje por Refuerzo (UVG, ciclo 2, 2026). Simulamos en SUMO un corredor de Av. La Reforma (zona 9/10, 5 semáforos) y un cruce recortado de él, con el fin de entrenar agentes RL que controlen los semáforos y compararlos contra tres líneas base sin RL.

Este repositorio contiene el **entorno de simulación** y las **líneas base**: la red de SUMO calibrada, la demanda por escenario, la fábrica de entornos (Gymnasium para un agente y PettingZoo para multiagente, vía `sumo-rl`) y los tres controladores base —tiempo fijo, actuado y max-pressure— sobre los que se construyen los agentes RL.

Guía rápida: [instalar](#instalación) -> [reconstruir la red](#reconstruir-la-red-y-la-demanda) -> [escenarios](#escenarios) -> [líneas base](#líneas-base) -> [pruebas](#pruebas).

## Instalación
Necesita Python 3.11 (SUMO, libsumo y sumo-rl se instalan con pip, sin instalar SUMO aparte).

```bash
bash scripts/setup_env.sh      
source scripts/env.sh          
python -m pytest -q            
```
- Con `uv` instalado, `setup_env.sh` crea el venv con Python 3.11; si no, usa `python3 -m venv`. `VENV=.venv bash scripts/setup_env.sh` lo deja dentro del repo.
- `pettingzoo==1.24.3` queda fijo en `requirements.txt` porque `sumo-rl` 1.4.5 se rompe con la 1.25 o posterior.
- Todo se corre desde la raíz del repo y después de `source scripts/env.sh`.

## Reconstruir la red y la demanda
La red, los programas fijo y actuado y los nombres de los semáforos ya vienen en `env/net/`. Para regenerarlos desde el extracto de OpenStreetMap y los conteos:
```bash
bash scripts/build_network.sh   
python -m env.validate          
```
La demanda de cada episodio se genera al vuelo con la semilla del episodio (`env/demand.py`, caché en `env/routes/cache/`). Detalle de la red y de la calibración: `env/net/README.md` y `env/conteos/README.md`.

## Escenarios
Los escenarios se piden con `get_scenario("reforma_pico")` (`env/scenarios.py`). Hay dos redes —el corredor completo (`reforma_*`, para multiagente) y el cruce más cargado recortado de él (`cruce_*`, para un agente)— combinadas con cuatro perfiles de demanda:

| Perfil | Uso |
|---|---|
| `pico` | hora pico (entrenamiento) |
| `mediodia` | mediodía (hora no vista) |
| `pico_x13` | hora pico +30 % (estrés) |
| `noche` | noche = pico × 0.3 |

Así, por ejemplo, `reforma_pico` es el corredor en hora pico y `cruce_mediodia` el cruce a mediodía. La demanda es estocástica por semilla. También hay un `toy_grid2x2` (la rejilla 2×2 de `sumo-rl`) para pruebas rápidas.

## Líneas base
`baselines/` implementa los tres controladores sin RL, cada uno como una `Policy` expuesta con `make_policy(cfg)` y registrada en `agents/registry.py`:
- **Tiempo fijo** (`fixed_time.py`): el ciclo de reparto de Webster.
- **Actuado** (`actuated.py`): el semáforo `actuated` de SUMO (extiende el verde si detecta autos).
- **Max-pressure** (`max_pressure.py`): elige la fase que maximiza la presión por ocupación, respetando `max_green`.

## Pruebas
```bash
python -m pytest -q                 
python -m pytest -q -m "not slow"   
```

## Estructura
Las interfaces entre módulos están en [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md); el plan completo del proyecto, en [docs/PLAN.md](docs/PLAN.md).

| Carpeta | Contenido |
|---|---|
| `env/` | Red de Reforma (`net/`), conteos (`conteos/`), demanda (`demand.py`), escenarios y semillas (`scenarios.py`), fábrica de entornos (`reforma_env.py`), incidentes y validación |
| `baselines/` | Tiempo fijo, actuado, max-pressure |
| `agents/` | Interfaz común de las políticas (`base.py`) y registro de algoritmos (`registry.py`) |
| `scripts/` | Entorno (`setup_env.sh`, `env.sh`) y reconstrucción de la red (`build_network.sh`) |
| `docs/` | `PLAN.md` y `ARCHITECTURE.md` |
| `tests/` | Pruebas con pytest del entorno y las líneas base |

## Decisiones de diseño
- **Conteos sintéticos, PROVISIONALES.** `env/conteos/{pico,mediodia}.csv` no son conteos de campo: los genera `env/synthetic_counts.py` y se calibraron contra un tiempo de viaje de referencia que tampoco es medido (330 s hacia el norte y 300 s hacia el sur, un placeholder en `env/validate.py`). El ciclo del tiempo fijo es igual de sintético (reparto de Webster). Con datos reales se reemplazan los CSV y se reconstruye con `scripts/build_network.sh`, que no pisa conteos que ya no sean sintéticos.
- **Max-pressure es una línea base fuerte.** Usa conjuntos de carriles (no enlaces), presión por ocupación y hace cumplir `max_green`. Es la referencia principal a la que los agentes RL tendrán que ganarle, no solo al tiempo fijo.
