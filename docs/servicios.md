# Mapa de Microservicios y Dockerfiles — Online Boutique

Este documento analiza la arquitectura de contenedores del monorepo políglota de **Online Boutique**, documentando los lenguajes, frameworks, puertos, etapas de compilación multi-stage e imágenes base utilizadas por cada servicio para alimentar la matriz del pipeline de CI/CD (**P2**).

---

## 1. Tabla Resumen de Microservicios

| # | Microservicio | Lenguaje / Runtime | Directorio | Dockerfile Path | Imagen Base Builder | Imagen Base Runtime | Puerto | Protocolo |
|---|---|---|---|---|---|---|---|---|
| **1** | `frontend` | Go 1.27 | `src/frontend` | `src/frontend/Dockerfile` | `golang:1.27.0-alpine` | `gcr.io/distroless/static` | 8080 | HTTP / gRPC client |
| **2** | `cartservice` | C# (.NET 10) | `src/cartservice` | `src/cartservice/src/Dockerfile` | `mcr.microsoft.com/dotnet/sdk:10.0.100-noble` | `mcr.microsoft.com/dotnet/runtime-deps:10.0.0-noble-chiseled` | 7070 | gRPC |
| **3** | `productcatalogservice` | Go 1.27 | `src/productcatalogservice` | `src/productcatalogservice/Dockerfile` | `golang:1.27.0-alpine` | `gcr.io/distroless/static` | 3550 | gRPC |
| **4** | `currencyservice` | Node.js 24 | `src/currencyservice` | `src/currencyservice/Dockerfile` | `node:24.20.0-alpine` | `alpine:3.24.1` (con `nodejs`) | 7000 | gRPC |
| **5** | `paymentservice` | Node.js 24 | `src/paymentservice` | `src/paymentservice/Dockerfile` | `node:24.20.0-alpine` | `alpine:3.24.1` (con `nodejs`) | 50051 | gRPC |
| **6** | `shippingservice` | Go 1.27 | `src/shippingservice` | `src/shippingservice/Dockerfile` | `golang:1.27.0-alpine` | `gcr.io/distroless/static` | 50051 | gRPC |
| **7** | `emailservice` | Python 3.14 | `src/emailservice` | `src/emailservice/Dockerfile` | `python:3.14.7-alpine` | `python:3.14.7-alpine` | 8080 | gRPC |
| **8** | `checkoutservice` | Go 1.27 | `src/checkoutservice` | `src/checkoutservice/Dockerfile` | `golang:1.27.0-alpine` | `gcr.io/distroless/static` | 5050 | gRPC |
| **9** | `recommendationservice` | Python 3.14 | `src/recommendationservice` | `src/recommendationservice/Dockerfile` | `python:3.14.7-alpine` | `python:3.14.7-alpine` | 8080 | gRPC |
| **10** | `adservice` | Java (Temurin 25) | `src/adservice` | `src/adservice/Dockerfile` | `eclipse-temurin:25.0.4_7-jdk-noble` | `eclipse-temurin:25.0.4_7-jre-alpine` | 9555 | gRPC |
| **11** | `loadgenerator` | Python 3.14 (Locust) | `src/loadgenerator` | `src/loadgenerator/Dockerfile` | `python:3.14.7-alpine` | `python:3.14.7-alpine` | N/A (Egress) | HTTP client |
| *(ext)* | `shoppingassistantservice` | Python 3.14 (Gemini AI) | `src/shoppingassistantservice` | `src/shoppingassistantservice/Dockerfile` | `python:3.14.7-slim` | `python:3.14.7-slim` | 8080 | HTTP / REST |

---

## 2. Detalle de Arquitectura por Servicio

### 1. `frontend` (Go)
* **Función:** Interfaz web para el usuario y servidor de agregación. Expone rutas web HTML y se comunica internamente vía gRPC con el resto de servicios.
* **Estrategia Multi-Stage:**
  1. `builder`: `golang:1.27.0-alpine`. Descarga dependencias con `go mod download` y compila un binario estático optimizado con flags `CGO_ENABLED=0 go build -ldflags="-s -w"`.
  2. `runtime`: `gcr.io/distroless/static`. Copia únicamente el binario y los directorios `/templates` y `/static`. Sin shell, sin gestor de paquetes ni binarios del sistema, minimizando drásticamente la superficie de ataque.

### 2. `cartservice` (C# .NET)
* **Función:** Almacena los artículos añadidos al carrito por cada cliente. Soporta persistencia en Redis o fallback a almacenamiento en memoria volátil.
* **Estrategia Multi-Stage:**
  1. `builder`: `mcr.microsoft.com/dotnet/sdk:10.0.100-noble`. Restaura paquetes NuGet y publica con optimizaciones agresivas: `dotnet publish -p:PublishSingleFile=true -p:PublishTrimmed=true -p:TrimMode=full --self-contained true -c release`.
  2. `runtime`: `mcr.microsoft.com/dotnet/runtime-deps:10.0.0-noble-chiseled`. Imagen *chiseled* basada en Ubuntu sin utilidades de shell, ejecutando con usuario sin privilegios `USER 1000`.

### 3. `productcatalogservice` (Go)
* **Función:** Provee la lista de productos disponibles, búsqueda por palabras clave y detalle de ítems leyendo desde un archivo `products.json`.
* **Estrategia Multi-Stage:**
  1. `builder`: `golang:1.27.0-alpine`. Compilación estática sin CGO.
  2. `runtime`: `gcr.io/distroless/static`. Incluye exclusivamente el ejecutable y el archivo estático `products.json`.

### 4. `currencyservice` (Node.js)
* **Función:** Realiza conversión de monedas entre divisas internacionales basándose en tipos de cambio calculados.
* **Estrategia Multi-Stage:**
  1. `builder`: `node:24.20.0-alpine`. Requiere herramientas de compilación (`python3`, `make`, `g++`) para módulos nativos de Node.js durante `npm install --only=production`.
  2. `runtime`: `alpine:3.24.1` con paquete `nodejs` en ejecución limpia. No arrastra herramientas de compilación ni librerías sobrantes.

### 5. `paymentservice` (Node.js)
* **Función:** Valida y procesa información de tarjetas de crédito simuladas, autorizando transacciones.
* **Estrategia Multi-Stage:**
  1. `builder`: `node:24.20.0-alpine`. Instala dependencias productivas limpias.
  2. `runtime`: `alpine:3.24.1` con runtime Node.js ligero.

### 6. `shippingservice` (Go)
* **Función:** Calcula cotizaciones de costos de envío basadas en el número de ítems y simula la generación de números de rastreo.
* **Estrategia Multi-Stage:**
  1. `builder`: `golang:1.27.0-alpine`.
  2. `runtime`: `gcr.io/distroless/static`.

### 7. `emailservice` (Python)
* **Función:** Simula el envío de correos electrónicos transaccionales de confirmación de compra a los clientes.
* **Estrategia Multi-Stage:**
  1. `base`: `python:3.14.7-alpine`.
  2. `builder`: Instala cabeceras del kernel (`linux-headers`) y `g++` para instalar paquetes wheel con extensiones C/C++.
  3. `runtime`: Copia `/usr/local/lib/python3.14/` a la capa base, instalando solo `libstdc++` para ejecución.

### 8. `checkoutservice` (Go)
* **Función:** Orquestador central de la compra. Coordina llamadas gRPC a `cartservice`, `shippingservice`, `paymentservice`, `emailservice` y `productcatalogservice`.
* **Estrategia Multi-Stage:**
  1. `builder`: `golang:1.27.0-alpine`.
  2. `runtime`: `gcr.io/distroless/static`.

### 9. `recommendationservice` (Python)
* **Función:** Genera una lista de recomendaciones de productos similares basados en los ítems actualmente en el carrito de compra.
* **Estrategia Multi-Stage:**
  1. `builder`: Instala dependencias y gRPC bindings.
  2. `runtime`: Alpine minimalista con Python 3.14 y runtime libraries.

### 10. `adservice` (Java)
* **Función:** Devuelve anuncios publicitarios contextuales en base a las etiquetas de los productos visualizados.
* **Estrategia Multi-Stage:**
  1. `builder`: `eclipse-temurin:25.0.4_7-jdk-noble` con JDK completo para descargar dependencias de repositorios y empaquetar la distribución con Gradle (`gradlew installDist`).
  2. `runtime`: `eclipse-temurin:25.0.4_7-jre-alpine`. Ejecuta exclusivamente con el Java Runtime Environment (JRE) sobre Alpine Linux.

### 11. `loadgenerator` (Python / Locust)
* **Función:** Generador de carga sintética continua que simula clientes navegando, añadiendo artículos y realizando checkout contra `frontend`.
* **Estrategia Multi-Stage:**
  1. `builder`: Instala la suite de testing de rendimiento Locust en un prefijo aislado (`--prefix="/install"`).
  2. `runtime`: Copia `/install` a `/usr/local` y ejecuta en modo *headless* sin interfaz web para consumo eficiente de recursos.

### *(Ext)* `shoppingassistantservice` (Python / Gemini)
* **Función:** Servicio opcional que provee asistencia de compras mediante LLM (Gemini) sobre peticiones HTTP.
* **Estrategia Multi-Stage:**
  1. `builder`: `python:3.14.7-slim` con Debian packages de compilación.
  2. `runtime`: Copia paquetes limpios y expone puerto 8080.

---

## 3. Implicaciones Clave para el Pipeline de CI/CD (P2)

1. **Ruta no estándar en `cartservice`:** A diferencia del resto de servicios donde el `Dockerfile` reside en la raíz de la carpeta del servicio (`src/<service>/Dockerfile`), en `cartservice` reside en `src/cartservice/src/Dockerfile`, y su contexto de compilación debe ser `src/cartservice/src`. El pipeline de paths-filter y Docker buildx debe manejar esta particularidad.
2. **Uso de imágenes distroless y chiseled:** La gran mayoría de los microservicios (`frontend`, `checkoutservice`, `productcatalogservice`, `shippingservice`, `cartservice`) usan imágenes base de producción libres de shell (`distroless/static` o `noble-chiseled`). Esto reduce el número de vulnerabilidades reportadas por Trivy en **P2-04** a prácticamente cero en el sistema operativo base.
3. **Optimización con GHA Cache en Buildx:** Al ser un monorepo políglota, la descarga de dependencias (Go modules, paquetes NuGet, npm, pip, Gradle dependencies) se acelerará significativamente en **P2-02** usando `cache-from: type=gha` y `cache-to: type=gha,mode=max`.
