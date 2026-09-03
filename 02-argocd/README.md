# Fase 2: Argo CD — Operador GitOps para Kubernetes

## 2.1 ¿Qué es Argo CD?

Argo CD es un **operador GitOps declarativo para Kubernetes**. Monitorea continuamente un repositorio Git y asegura que el estado del clúster refleja exactamente lo que está en Git.

### Conceptos clave de Argo CD

| Concepto | Descripción |
|---|---|
| **Application** | Objeto de Argo CD que conecta un repositorio Git con un destino en Kubernetes |
| **Sync** | Proceso de aplicar el estado de Git al clúster |
| **OutOfSync** | Estado cuando Git y el clúster difieren |
| **Synced** | Estado cuando Git y el clúster son idénticos |
| **Self-healing** | Capacidad de revertir cambios manuales no autorizados en el clúster |

---

## 2.2 Instalación de Argo CD

### Paso 1: Instalar Argo CD

```bash
# Añadir el repositorio oficial de Argo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Instalar Argo CD con valores por defecto
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --wait

```

```bash
# Observar la creación de pods en tiempo real
kubectl get pods -n argocd --watch
```
**Espera hasta que TODOS los pods estén `Running`**

### Paso 2: Verificar los CRDs de Argo CD

```bash
# Argo CD instala sus propios CRDs
kubectl get crds | grep argoproj
```

Deberías ver:
```
applications.argoproj.io
applicationsets.argoproj.io
appprojects.argoproj.io
```

---

## 2.3 Acceder a la UI de Argo CD

Argo CD incluye una interfaz web completa. Para acceder desde un clúster Kind necesitamos hacer port-forwarding.

### Paso 1: Exponer el servidor de Argo CD

```bash
# En una terminal separada (déjala corriendo)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

> Deja esta terminal abierta durante toda la PoC. Puedes abrir una nueva terminal para los siguientes pasos.

### Paso 2: Obtener la contraseña inicial

```bash
# La contraseña inicial del usuario 'admin' está en un Secret
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Guarda esta contraseña — la necesitarás para el login.

### Paso 3: Acceder a la UI

Abre tu navegador en: **https://localhost:8080**

**Credenciales:**
- Usuario: `admin`
- Contraseña: (la que obtuviste en el paso anterior)

### Paso 4: Instalar el CLI de Argo CD

```bash
# macOS con Homebrew
brew install argocd

# Verificar instalación
argocd version --client
```

### Paso 5: Login desde el CLI

```bash
# Login al servidor (acepta el certificado con --insecure)
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure

# Verificar el login
argocd account list
```

---

## 2.4 Preparar los manifiestos en Git

Antes de crear la Application en Argo CD, todos los manifiestos de Crossplane deben estar commiteados en tu repositorio. Argo CD los leerá desde ahí.

### Estructura esperada en el repositorio

```
tu-repo/
└── 01-crossplane/
    ├── provider.yaml        ← Provider de PostgreSQL
    ├── provider-config.yaml ← Credenciales de conexión
    ├── function.yaml        ← Función patch-and-transform
    ├── xrd.yaml             ← Define la API XPostgreSQLInstance
    ├── composition.yaml     ← Cómo se materializan los recursos
    └── claim.yaml           ← Solicita la creación de la base de datos
```


```bash
# Desde la raíz del repositorio
git add 01-crossplane/
git commit -m "feat: add crossplane manifests for gitops management"
git push origin main
```

---

## 2.5 Conectar Argo CD al repositorio

**Repositorio público:**

```bash
argocd repo add https://github.com/TU-USUARIO/TU-REPOSITORIO.git
argocd repo list
```

**Repositorio privado (con Personal Access Token):**

```bash
# GitHub → Settings → Developer Settings → Personal Access Tokens
argocd repo add https://github.com/TU-USUARIO/TU-REPOSITORIO.git \
  --username TU-USUARIO-GITHUB \
  --password TU-PERSONAL-ACCESS-TOKEN

argocd repo list
```

En la UI: **Settings → Repositories** → verifica que el estado sea `Successful`.

---

## 2.6 Crear la Application de Argo CD

Una `Application` conecta una fuente en Git con un destino en Kubernetes.

Modifica el archivo `02-argocd/app-gitops-demo.yaml`, commitealo y aplícalo:

```bash
# Una vez creado el archivo YAML en 02-argocd/app-gitops-demo.yaml:
git add 02-argocd/app-gitops-demo.yaml
git commit -m "feat: add ArgoCD Application for crossplane infra"
git push origin main

# Aplicar el manifiesto directamente al clúster
kubectl apply -f 02-argocd/app-gitops-demo.yaml
```

---

## 2.7 Observar la reconciliación GitOps

### Verificar desde el CLI

```bash
# Ver el estado general de la Application
argocd app get gitops-demo

# Ver todos los recursos gestionados por Argo CD
argocd app resources crossplane-infra
```

### Confirmar la base de datos en PostgreSQL

```bash
# El Database Managed Resource debe estar SYNCED=True y READY=True
kubectl get databases.postgresql.postgresql.upbound.io

# Conectarse a PostgreSQL (con port-forward activo en otra terminal)
PGPASSWORD=platform123 psql -h localhost -U postgres -c "\l"
# Deberías ver 'appdb' en la lista
```

---

## 2.8 El ciclo GitOps completo

Ahora Argo CD es la única fuente de verdad. Para cualquier cambio en la infraestructura:

1. Edita el manifiesto en tu repositorio local
2. Haz commit y push
3. Argo CD detecta el cambio (polling cada 3 min, o instantáneo con webhook)
4. Argo CD aplica el diff al clúster
5. Crossplane reconcilia y ejecuta el cambio real en PostgreSQL

**Prueba práctica:** Modifica el `claim.yaml` para cambiar el nombre de la base de datos:

```bash
# Edita 01-crossplane/claim.yaml → spec.parameters.dbName: "appdb-v2"
git add 01-crossplane/claim.yaml
git commit -m "feat: rename database to appdb-v2"
git push origin main

# Forzar sincronización inmediata (sin esperar el polling)
argocd app sync crossplane-infra

# Observar
argocd app get crossplane-infra
kubectl get databases.postgresql.postgresql.upbound.io
```

---


*Continúa con → [Fase 3: Integración GitOps](../03-integracion/GUIA_INTEGRACION.md)*
