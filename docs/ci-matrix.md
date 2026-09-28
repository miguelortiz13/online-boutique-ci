# Pipeline de Detección de Cambios y Matriz Dinámica — P2-02

Este documento describe la arquitectura y los resultados de medición del pipeline de integración continua implementado para compilar microservicios políglotas en **Online Boutique** ([`online-boutique-ci`](https://github.com/miguelortiz13/online-boutique-ci)).

---

## 1. Arquitectura del Flujo CI (`.github/workflows/ci.yaml`)

El pipeline consta de dos etapas secuenciales (`changes` ➔ `build`):

```mermaid
flowchart TD
    Trigger["PR / Push a main / workflow_dispatch"] --> JobChanges["Job changes<br/>(dorny/paths-filter@v3)"]
    
    subgraph Deteccion ["1. Detección Inteligente"]
        JobChanges --> Filter["Evalúa rutas src/*/**<br/>Produce JSON: ['frontend', ...]"]
        Filter --> Fallback{"¿Rutas modificadas?"}
        Fallback -->|Sí| Out["Output: services = ['...']"]
        Fallback -->|No (cambio global / workflow)| Def["Default: ['frontend'] para validación"]
    end
    
    subgraph Matriz ["2. Compilación Paralela Dinámica"]
        Out --> MatrixJob["Job build (fromJSON(services))"]
        Def --> MatrixJob
        MatrixJob --> Resolver["Resuelve contexto y Dockerfile<br/>(Normal: ./src/<svc>, Cartservice: ./src/cartservice/src)"]
        Resolver --> Buildx["docker/build-push-action@v6<br/>cache-from: type=gha,scope=<svc><br/>cache-to: type=gha,mode=max,scope=<svc>"]
        Buildx --> TestImg["docker images (Verificación local)"]
    end
```

---

## 2. Decisiones Técnicas Clave

1. **Aislamiento por Scope en GitHub Actions Cache:**
   Al compilar hasta 12 servicios distintos en un monorepo, un caché plano `type=gha` provocaría sobreescrituras constantes y desalojo de capas (*cache thrashing*). Configuramos `scope=${{ matrix.service }}` para que cada microservicio mantenga su propio ciclo de vida de caché independiente dentro del límite de 10 GB del repositorio.
2. **Normalización de Contextos para `cartservice`:**
   Dado que `cartservice` organiza su código fuente dentro de `src/cartservice/src/Dockerfile` y su solución `.sln` en la raíz, el paso `Resolve Context & Dockerfile` ajusta dinámicamente el `context` y `file` antes de invocar `docker/build-push-action`.
3. **Optimización FinOps en CI:**
   En lugar de compilar los 11 servicios en cada Pull Request (lo que tomaría ~15-20 minutos de runner por commit), `dorny/paths-filter` detecta con precisión quirúrgica el subconjunto modificado. Si solo cambia `src/frontend`, el cómputo se reduce en más del **90%**.

---

## 3. Evidencia y Métricas de Rendimiento (Benchmark con/sin Caché)

Se realizaron pruebas reales en GitHub Actions con Pull Requests sucesivos para validar la detección y medir el impacto del caché:

| Prueba / Ejecución | Evento / Rama | Servicios Compilados | Tiempo de Detección | Tiempo de Build (`frontend`) | Caché GHA | Resultado |
|---|---|:---:|:---:|:---:|:---:|:---:|
| **PR #1 (Frío)** | PR `#1` (`feat/ci-paths-filter`) | `frontend` (1) | 5s | **1m 51s** | ❌ Miss (Inicialización) | [Run #36377839956](https://github.com/miguelortiz13/online-boutique-ci/actions/runs/36377839956) |
| **Push a `main`** | Merge PR `#1` | `frontend` (1) | 6s | **1m 30s** | 💾 Exporta caché a `main` | [Run #36378010487](https://github.com/miguelortiz13/online-boutique-ci/actions/runs/36378010487) |
| **PR #2 (Caliente)** | PR `#2` (`test/frontend-cache-test`) | `frontend` (1) | 5s | **1m 12s** | ✅ **HIT** (`#10`, `#11`, `#15` `CACHED`) | [Run #36378188338](https://github.com/miguelortiz13/online-boutique-ci/actions/runs/36378188338) |

### Conclusiones del Benchmark:
* **Ahorro de Tiempo con Caché:** El build con capas cacheadas redujo el tiempo de compilación de **1m 51s a 1m 12s** (**~35% de aceleración** en un microservicio Go individual).
* **Cumplimiento de Aceptación:** Al modificar únicamente `src/frontend/main.go`, el job `changes` filtró los otros 10 microservicios, ejecutando una matriz de exactamente **1 solo servicio** (`Build (frontend)`), cumpliendo el criterio canónico de **P2-02**.
