# Promoción Automatizada a Repositorio GitOps mediante Kustomize y PRs (P2-06)

## 1. Contexto y Principios de GitOps

En una arquitectura de entrega continua moderna para entornos nativos de la nube, **la integración continua (CI) debe estar estrictamente desacoplada del despliegue continuo (CD)**:

* **Responsabilidad de CI (`online-boutique-ci`):** Compilar el código fuente, ejecutar pruebas unitarias, validar buenas prácticas de contenedor (Hadolint), generar el inventario SBOM (Syft), escanear vulnerabilidades (Trivy), publicar imágenes inmutables en Azure Container Registry (ACR) y firmarlas criptográficamente (Cosign Keyless via Sigstore).
* **Responsabilidad de CD (`platform-gitops`):** Mantener el estado deseado declarativo del clúster de Kubernetes en Git como única fuente de verdad (*Single Source of Truth*), gestionado y reconciliado de manera continua por operadores GitOps (como Argo CD o Flux).

### Beneficios del Patrón de Promoción por Pull Request:
1. **Trazabilidad y Auditoría:** Cada cambio de versión en un microservicio queda registrado como un commit y un Pull Request con autor, hash de commit origen, digest inmutable y enlace a la ejecución de CI.
2. **Control de Puertas de Enlace (Gating):** Permite revisión humana o validaciones automáticas de políticas antes del despliegue en entornos productivos.
3. **Mínimo Privilegio (PoLP):** El pipeline de CI nunca requiere acceso de red ni credenciales hacia el clúster de Kubernetes; únicamente necesita permisos para abrir Pull Requests en el repositorio GitOps.
4. **Capacidad de Rollback Inmediata:** Revertir un despliegue se reduce a hacer un `git revert` en el repositorio GitOps.

---

## 2. Diagrama de Arquitectura de la Promoción

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Desarrollador
    participant CI as GitHub Actions (online-boutique-ci)
    participant ACR as Azure Container Registry
    participant Sig as Sigstore / Cosign
    participant GitOps as GitHub (platform-gitops)
    participant Argo as Argo CD / K8s Cluster (P3)

    Dev->>CI: Git Push a rama main
    CI->>CI: Lint (Hadolint) + Tests (Go/.NET)
    CI->>ACR: Buildx + Push Imagen (sha-<short>)
    CI->>Sig: Firma Keyless + Atestación SBOM
    Note over CI,GitOps: Inicio del paso de Promoción GitOps (P2-06)
    CI->>GitOps: Clona repo platform-gitops
    CI->>GitOps: kustomize edit set image acr.../servicio:sha-<short>
    CI->>GitOps: Crea rama promote/<servicio>-<tag> y commit
    CI->>GitOps: Abre Pull Request contra main con metadata completa
    Note over GitOps,Argo: En P3: Aprobación / Merge y Reconciliación
    GitOps->>Argo: Webhook / Polling detecta nuevo commit en main
    Argo->>Argo: Despliega nueva versión en AKS
```

---

## 3. Estructura del Repositorio GitOps (`platform-gitops`)

El repositorio GitOps [`miguelortiz13/platform-gitops`](https://github.com/miguelortiz13/platform-gitops) sigue el estándar canónico de overlays de **Kustomize**:

```text
platform-gitops/
├── README.md
└── apps/
    └── online-boutique/
        ├── base/
        │   ├── deployment-frontend.yaml
        │   └── kustomization.yaml
        └── overlays/
            ├── dev/
            │   └── kustomization.yaml     <-- Actualizado automáticamente por CI
            └── prod/
                └── kustomization.yaml    <-- Promovido mediante tags de release
```

### Configuración en `overlays/dev/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: online-boutique

resources:
  - ../../base

images:
  - name: acrdevsrelab01.azurecr.io/frontend
    newName: acrdevsrelab01.azurecr.io/frontend
    newTag: sha-10599a0
```

---

## 4. Implementación del Paso de Promoción en GitHub Actions

En `.github/workflows/ci.yaml`, tras completar la firma y verificación criptográfica con Cosign, se ejecuta el paso de promoción:

```yaml
- name: Set up Kustomize
  if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
  run: |
    if ! command -v kustomize &> /dev/null; then
      echo "Instalando kustomize oficial..."
      curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
      sudo mv kustomize /usr/local/bin/
    fi
    kustomize version

- name: Promote to GitOps (platform-gitops)
  if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
  env:
    GH_TOKEN: ${{ secrets.GITOPS_PROMOTION_TOKEN }}
    SERVICE: ${{ matrix.service }}
    IMAGE_TAG: ${{ steps.push-image.outputs.image_tag }}
    IMAGE_DIGEST: ${{ steps.push-image.outputs.image_digest }}
    ACR_NAME: ${{ vars.ACR_NAME }}
  run: |
    if [ -z "$GH_TOKEN" ]; then
      echo "::warning::GITOPS_PROMOTION_TOKEN no configurado; omitiendo promoción GitOps."
      exit 0
    fi

    echo "Configurando identidad git..."
    git config --global user.name "github-actions[bot]"
    git config --global user.email "github-actions[bot]@users.noreply.github.com"

    GITOPS_REPO="miguelortiz13/platform-gitops"
    TEMP_DIR="$(mktemp -d)"

    echo "Clonando ${GITOPS_REPO}..."
    git clone "https://x-access-token:${GH_TOKEN}@github.com/${GITOPS_REPO}.git" "${TEMP_DIR}"
    cd "${TEMP_DIR}/apps/online-boutique/overlays/dev"

    ACR_REGISTRY="${ACR_NAME}.azurecr.io"
    TARGET_IMAGE="${ACR_REGISTRY}/${SERVICE}:${IMAGE_TAG}"

    echo "Actualizando kustomization para ${SERVICE} a ${TARGET_IMAGE}..."
    kustomize edit set image "${ACR_REGISTRY}/${SERVICE}=${TARGET_IMAGE}"

    if git diff --quiet kustomization.yaml; then
      echo "No hay cambios en kustomization.yaml para dev. Saliendo sin cambios."
      exit 0
    fi

    BRANCH="promote/${SERVICE}-${IMAGE_TAG}"
    echo "Creando rama ${BRANCH}..."
    git checkout -b "${BRANCH}"
    git add kustomization.yaml
    git commit -m "chore(gitops): promote ${SERVICE} to ${IMAGE_TAG} [skip ci]"

    echo "Pusheando rama ${BRANCH} a origin..."
    git push -u origin "${BRANCH}" --force

    EXISTING_PR=$(gh pr list --repo "${GITOPS_REPO}" --head "${BRANCH}" --json number -q '.[0].number')
    if [ -n "$EXISTING_PR" ]; then
      echo "Pull Request ya existente: #${EXISTING_PR}"
    else
      echo "Creando Pull Request en ${GITOPS_REPO}..."
      gh pr create \
        --repo "${GITOPS_REPO}" \
        --base main \
        --head "${BRANCH}" \
        --title "chore(gitops): promote ${SERVICE} to ${IMAGE_TAG}" \
        --body "..."
    fi
```

---

## 5. Gestión Segura de Credenciales

* El secreto `GITOPS_PROMOTION_TOKEN` reside en `miguelortiz13/online-boutique-ci`.
* Posee permisos estrictos de escritura sobre el repositorio `platform-gitops` (`repo` scope o fine-grained token `contents:write`, `pull-requests:write`).
* No tiene acceso de administración al clúster Kubernetes ni credenciales de infraestructura Azure, aislando el radio de explosión (*blast radius*) de CI.
