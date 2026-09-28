# Linting de Dockerfiles y Tests Unitarios en CI — P2-03

Este documento detalla la estrategia de *Shift-Left Testing* y análisis estático de seguridad para los contenedores de **Online Boutique** ([`online-boutique-ci`](https://github.com/miguelortiz13/online-boutique-ci)), garantizando que los defectos y malas prácticas se detecten antes de consumir recursos en la etapa de compilación (`build`).

---

## 1. Arquitectura de Dependencias del Pipeline

```mermaid
flowchart TD
    Trigger["PR / Push a main / manual"] --> Changes["Job: Detect Changed Services<br/>(dorny/paths-filter)"]

    subgraph ShiftLeft ["Shift-Left Validation (Paralelo)"]
        Changes --> Lint["Job: Lint Dockerfile<br/>(hadolint/hadolint-action@v3.1.0)"]
        Changes --> Test["Job: Unit Tests<br/>(go test -v ./... / dotnet test)"]
    end

    subgraph BuildStage ["Compilación y Empaquetado"]
        Lint -->|Success| Build["Job: Build Container Image<br/>(docker/build-push-action@v6)"]
        Test -->|Success| Build
        Lint -.->|Failure (Bad Practice)| Blocked["⛔ Build Bloqueado"]
        Test -.->|Failure (Test Roto)| Blocked
    end
```

---

## 2. Configuración de Hadolint (`.hadolint.yaml`)

Se implementó el archivo de configuración [`.hadolint.yaml`](.hadolint.yaml) con umbral de fallo en `warning`:

```yaml
failure-threshold: warning
ignored:
  - DL3018 # Pin versions in apk add (gestionado en actualizaciones de imágenes base)
  - DL3008 # Pin versions in apt-get install (gestionado en actualizaciones de imágenes base)
  - DL3042 # Avoid use of cache directory with pip (manejado en builds multi-stage)
  - DL3045 # COPY to a relative destination without WORKDIR set
  - DL3025 # Use arguments JSON notation for CMD and ENTRYPOINT arguments (loadgenerator)
  - DL3059 # Multiple consecutive RUN instructions
```

### Reglas estrictas aplicadas:
* **`DL3006`:** Prohíbe imágenes base sin tag o no versionadas explícitamente (se migraron los servicios Go de `FROM gcr.io/distroless/static` a `FROM gcr.io/distroless/static:nonroot`).
* **`DL3007`:** Prohíbe el uso de `:latest` en imágenes base para evitar *drifts* y roturas imprevistas.
* **`DL3002`:** Prohíbe finalizar la imagen con el usuario `root`.

---

## 3. Pruebas Unitarias Integradas

El job `test` detecta automáticamente el runtime de cada servicio en la matriz:
* **Microservicios Go (`frontend`, `productcatalogservice`, `shippingservice`, `checkoutservice`):** Configuración automática de Go 1.27 vía `actions/setup-go@v5` y ejecución de `go test -v ./...`.
* **Microservicio C# (`cartservice`):** Configuración de .NET 10 vía `actions/setup-dotnet@v4` y ejecución de `dotnet test src/cartservice/tests/`.
* **Servicios sin tests:** Mensaje informativo y paso limpio para no bloquear el flujo.

---

## 4. Evidencia y Validación Experimental

### Caso 1: Flujo Exitoso (PR #3)
* **Pull Request:** [#3 (`feat/ci-lint-and-tests`)](https://github.com/miguelortiz13/online-boutique-ci/pull/3)
* **Run:** [Run #36378877847](https://github.com/miguelortiz13/online-boutique-ci/actions/runs/36378877847)
* **Resultados:**
  - `Lint Dockerfile` pasó en todos los servicios modificados (`frontend`: 8s, `checkoutservice`: 7s, `shippingservice`: 11s, `productcatalogservice`: 14s).
  - `Unit Tests` pasaron al 100% (`shippingservice`: 34s, `checkoutservice`: 49s, `frontend`: 46s, `productcatalogservice`: 56s).
  - Al completar exitosamente ambos filtros, se desbloqueó y ejecutó el job `build` para los 4 servicios.

### Caso 2: Prueba Negativa de Mala Práctica (PR #4)
* **Pull Request:** [#4 (`test/hadolint-bad-practice`)](https://github.com/miguelortiz13/online-boutique-ci/pull/4)
* **Run:** [Run #36379196990](https://github.com/miguelortiz13/online-boutique-ci/actions/runs/36379196990)
* **Inyección de defecto:** Se introdujo la directiva prohibida `USER root` al final de `src/frontend/Dockerfile`.
* **Comportamiento:**
  - `hadolint` detectó `DL3002 warning: Last USER should not be root` en **8 segundos**.
  - El job `lint` falló inmediatamente con código de salida 1.
  - El job `build` fue **cancelado/bloqueado automáticamente (0s)**, demostrando el cumplimiento estricto del criterio de aceptación: *"Un Dockerfile con mala práctica hace fallar el PR"*.
