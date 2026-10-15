# Publicación en GitHub Container Registry (GHCR) y Firma Criptográfica con Cosign (P2-05)

> **Cambio de registro (2026-10):** las imágenes se publicaban en Azure Container Registry (`acrdevsrelab01`, SKU Basic, ~USD 5/mes fijos). Se movieron a GHCR, gratis para repos públicos, que autentica con el `GITHUB_TOKEN` del propio workflow. La firma keyless y la atestación del SBOM no cambian.

## 1. Contexto y Objetivos de Seguridad

En un modelo de seguridad Zero-Trust para entornos de producción en Kubernetes, **el clúster no debe confiar ciegamente en ninguna imagen** por el simple hecho de que se encuentre en un registro de contenedores. Para garantizar que únicamente los artefactos generados, testeados y validados por nuestro pipeline oficial de CI puedan ser admitidos y ejecutados (regla que se forzará en **P6** con **Kyverno**), se requiere:

1. **Autenticación sin Secretos (Secretless CI):** El push a GHCR usa el `GITHUB_TOKEN` efímero del job (válido solo durante la ejecución). No hay credenciales estáticas de larga vida.
2. **Principio de Menor Privilegio (PoLP):** El workflow solo pide `packages: write` (publicar en GHCR) e `id-token: write` (firma keyless); el resto de permisos son de lectura.
3. **Inmutabilidad y Trazabilidad:** Etiquetar las imágenes con el SHA corto del commit (`sha-<corto>`), publicar tags semánticos en releases (`vX.Y.Z`) y firmar siempre sobre el **digest inmutable** (`@sha256:...`), nunca sobre etiquetas mutables.
4. **Firma Criptográfica Keyless (Sigstore / Cosign):** Firmar digitalmente los artefactos utilizando certificados x509 efímeros emitidos por **Fulcio** basados en la identidad OIDC de GitHub Actions, auditados en el registro público de transparencia **Rekor**.
5. **Atestación de SBOM:** Vincular criptográficamente el inventario de software (SBOM SPDX 2.3 generado por Syft) como una atestación *in-toto* adjunta a la imagen en GHCR.

---

## 2. Arquitectura de Identidad y Publicación

```mermaid
sequenceDiagram
    autonumber
    participant GHA as GitHub Actions Runner
    participant OIDC as GitHub OIDC Provider
    participant GHCR as GitHub Container Registry
    participant Sigstore as Sigstore (Fulcio / Rekor)

    GHA->>GHCR: 1. Login con GITHUB_TOKEN (packages: write)
    GHA->>GHCR: 2. Push de la imagen (sha-<short>)
    GHCR-->>GHA: Retorna RepoDigest (sha256:...)
    GHA->>OIDC: 3. Solicita JWT OIDC (id-token: write)
    OIDC-->>GHA: Retorna JWT con claims del repositorio
    GHA->>Sigstore: 4. Firma Keyless con Cosign usando el JWT
    Sigstore-->>GHA: Certificado Fulcio + Registro en Rekor
    GHA->>GHCR: 5. Publica firma OCI (.sig) y atestación SBOM (.att)
```

---

## 3. Registro y Permisos

* **Registro:** `ghcr.io/miguelortiz13/online-boutique/<servicio>`, definido en la variable `REGISTRY` del workflow.
* **Vínculo con el repo:** la imagen lleva la etiqueta `org.opencontainers.image.source`, así el paquete aparece en el repo y hereda sus permisos.
* **Visibilidad:** los paquetes nuevos de una cuenta personal se crean privados. Se dejan **públicos** (Package settings → Change visibility) para que el clúster los descargue sin `imagePullSecrets`.
* **Permisos del workflow:** `packages: write` e `id-token: write`. No hace falta configurar variables ni secretos en el repositorio.

---

## 4. Estrategia de Etiquetado y Firma

### 1. Etiquetado
| Evento | Etiqueta(s) Publicada(s) | Propósito |
| :--- | :--- | :--- |
| **Pull Request / Branch** | `sha-<short>` | Identificador inmutable y reproducible por commit. |
| **Push a `main`** | `sha-<short>` + `latest` | Tag inmutable para auditoría y tag móvil para última versión estable. |
| **Release de versión** | `vX.Y.Z` | Versión semántica formal del software. |

### 2. Firma Keyless sobre el Digest
La firma **nunca se realiza sobre un tag mutable** (como `latest` o `v1.0.0`), ya que un atacante o error operacional podría mover la etiqueta hacia otra imagen no autorizada. Cosign firma el **digest inmutable**:

```bash
# Extracción del digest del manifiesto publicado
REPO_DIGEST=$(docker inspect --format='{{range .RepoDigests}}{{.}}{{"\n"}}{{end}}' "${IMAGE_NAME}:sha-${SHORT_SHA}" | grep "^${IMAGE_NAME}@" | head -n1)

# Firma keyless con Cosign
cosign sign --yes "${REPO_DIGEST}"
```

### 3. Atestación del SBOM (Software Bill of Materials)
El archivo SBOM generado por Syft en formato SPDX 2.3 JSON (`sbom-<servicio>.spdx.json`) se adjunta a la imagen en GHCR como una atestación criptográficamente firmada:

```bash
cosign attest --yes --predicate "sbom-${SERVICE}.spdx.json" --type spdx "${REPO_DIGEST}"
```

---

## 5. Guía de Verificación Local para Auditoría / SecOps

Cualquier ingeniero o sistema de auditoría puede verificar localmente la autenticidad e integridad de la imagen directamente desde GHCR (paquetes públicos: no hace falta login):

### 1. Verificar la firma de la imagen:
```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/miguelortiz13/online-boutique-ci/" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/miguelortiz13/online-boutique/frontend@sha256:<DIGEST>
```

**Resultado esperado:**
Cosign valida la cadena de certificados efímeros de Fulcio, comprueba la inclusión en el log de transparencia de Rekor y emite el payload JSON confirmando que la imagen fue construida y firmada exclusivamente por el pipeline de GitHub Actions del repositorio `miguelortiz13/online-boutique-ci`.

### 2. Verificar la atestación del SBOM:
```bash
cosign verify-attestation \
  --type spdx \
  --certificate-identity-regexp "https://github.com/miguelortiz13/online-boutique-ci/" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/miguelortiz13/online-boutique/frontend@sha256:<DIGEST>
```

---

## 6. Conexión con los Próximos Proyectos

* **P2-06:** Con la imagen publicada y firmada, el pipeline abrirá automáticamente un Pull Request hacia el repositorio `platform-gitops` actualizando la referencia de la imagen con el tag inmutable `sha-<short>`.
* **P6 (Seguridad Kubernetes):** El clúster contará con políticas de **Kyverno** que verificarán la firma `cosign` en el momento de la admisión del pod, rechazando cualquier contenedor que no cuente con una firma válida generada por este pipeline.
