# Análisis del Provider PostgreSQL

## Provider: `tages/provider-postgresql` v0.1.0

- **Paquete OCI:** `xpkg.upbound.io/tages/provider-postgresql:v0.1.0`
- **Código fuente:** [github.com/tagesjump/provider-postgresql](https://github.com/tagesjump/provider-postgresql)
- **Origen:** es un provider generado con **Upjet** a partir del provider de Terraform
  [`cyrilgdn/terraform-provider-postgresql`](https://registry.terraform.io/providers/cyrilgdn/postgresql/latest).
  Por eso cada Managed Resource se corresponde 1:1 con un recurso de Terraform
  (`postgresql_database`, `postgresql_role`, …).

---

## 1. Managed Resources disponibles

El provider registra **15 Managed Resources**, repartidos en 6 grupos de API.
Todos los grupos son subdominios de `postgresql.upbound.io` y todos están en la versión `v1alpha1`.

### Grupo `postgresql.postgresql.upbound.io/v1alpha1` (el principal)

| Kind | Recurso Terraform | Propósito |
|---|---|---|
| **`Database`** | `postgresql_database` | Crea y gestiona una **base de datos** dentro del servidor PostgreSQL (`CREATE DATABASE`). Es el recurso que usamos en la Composition. |
| **`Role`** | `postgresql_role` | Crea y gestiona un **rol/usuario** (`CREATE ROLE`): login, password, superuser, `CREATEDB`, límite de conexiones, fecha de expiración, etc. |
| **`Schema`** | `postgresql_schema` | Crea un **esquema** dentro de una base de datos (`CREATE SCHEMA`) y gestiona sus políticas de acceso. |
| **`Grant`** | `postgresql_grant` | Otorga **privilegios** (`GRANT`) sobre objetos existentes (tablas, secuencias, funciones, la propia base de datos…) a un rol. |
| **`Extension`** | `postgresql_extension` | Instala una **extensión** en una base de datos (`CREATE EXTENSION`), p. ej. `pgcrypto`, `postgis`, `uuid-ossp`. |
| **`Function`** | `postgresql_function` | Crea y gestiona una **función almacenada** (`CREATE FUNCTION`) con su cuerpo, lenguaje y argumentos. |
| **`Server`** | `postgresql_server` | Define un **foreign server** (`CREATE SERVER`) para el sistema de *Foreign Data Wrappers*. |
| **`Publication`** | `postgresql_publication` | Crea una **publicación** de replicación lógica (`CREATE PUBLICATION`): qué tablas se publican y qué operaciones. |
| **`Subscription`** | `postgresql_subscription` | Crea una **suscripción** de replicación lógica (`CREATE SUBSCRIPTION`) contra una publicación remota. |

### Otros grupos

| Kind | Grupo de API | Recurso Terraform | Propósito |
|---|---|---|---|
| **`Privileges`** | `default.postgresql.upbound.io/v1alpha1` | `postgresql_default_privileges` | Gestiona los **privilegios por defecto** (`ALTER DEFAULT PRIVILEGES`): permisos que heredarán automáticamente los objetos que se creen en el futuro. |
| **`Role`** | `grant.postgresql.upbound.io/v1alpha1` | `postgresql_grant_role` | Concede la **pertenencia de un rol a otro rol** (`GRANT rol_a TO rol_b`), con o sin `ADMIN OPTION`. Ojo: es un `Role` distinto al del grupo `postgresql.*`. |
| **`ReplicationSlot`** | `physical.postgresql.upbound.io/v1alpha1` | `postgresql_physical_replication_slot` | Crea un **slot de replicación física**, usado por réplicas en streaming. |
| **`Slot`** | `replication.postgresql.upbound.io/v1alpha1` | `postgresql_replication_slot` | Crea un **slot de replicación lógica** asociado a un plugin de decodificación. |
| **`Mapping`** | `user.postgresql.upbound.io/v1alpha1` | `postgresql_user_mapping` | Define el **mapeo de un usuario local a un usuario remoto** en un foreign server (`CREATE USER MAPPING`). |

> Comando que use verlos en el clúster una vez instalado el provider:
> ```bash
> kubectl get crds | grep postgresql.upbound.io
> ```

---

## 2. Campos requeridos del recurso `Database`

**apiVersion:** `postgresql.postgresql.upbound.io/v1alpha1` · **kind:** `Database` (recurso **cluster-scoped**).

### Requerido

En el CRD generado por Upjet **todos los campos de `spec.forProvider` están marcados como `Optional`** a nivel de
`kubebuilder`. La obligatoriedad real se impone con una regla **CEL** (`XValidation`) sobre el objeto:

```
!('*' in self.managementPolicies || 'Create' in self.managementPolicies || 'Update' in self.managementPolicies)
  || has(self.forProvider.name)
  || (has(self.initProvider) && has(self.initProvider.name))
```

Es decir: **el único campo obligatorio es `spec.forProvider.name`** (o su equivalente en `spec.initProvider.name`),
y solo se exige cuando el recurso va a crearse o actualizarse.

| Campo | Tipo | Descripción |
|---|---|---|
| `spec.forProvider.name` | `string` | Nombre de la base de datos. Debe ser único en el servidor PostgreSQL. **Único obligatorio.** |

Fuera de `forProvider`, a nivel de `spec` también es necesario en la práctica:

| Campo | Descripción |
|---|---|
| `spec.providerConfigRef.name` | Qué `ProviderConfig` usar. Si se omite se usa el llamado `default`; como el nuestro se llama `postgresql-config`, hay que indicarlo explícitamente. |

### Opcionales (`spec.forProvider`)

| Campo | Tipo | Por defecto | Descripción |
|---|---|---|---|
| `owner` | `string` | usuario de la conexión | Rol propietario de la base de datos. |
| `template` | `string` | `template0` | Base de datos plantilla. **Inmutable**: cambiarlo recrea el recurso. |
| `encoding` | `string` | `UTF8` | Codificación del juego de caracteres. **Inmutable**. |
| `lcCollate` | `string` | `C` | Orden de colación (`LC_COLLATE`). **Inmutable**. |
| `lcCtype` | `string` | `C` | Clasificación de caracteres (`LC_CTYPE`). **Inmutable**. |
| `tablespaceName` | `string` | `DEFAULT` | Tablespace por defecto de los objetos de la base de datos. |
| `connectionLimit` | `number` | `-1` (sin límite) | Máximo de conexiones concurrentes. |
| `allowConnections` | `boolean` | `true` | Si es `false`, nadie puede conectarse a la base de datos. |
| `isTemplate` | `boolean` | `false` | Permite que cualquier usuario con `CREATEDB` clone esta base de datos. |

Además existen los campos estándar de todo Managed Resource de Crossplane:
`spec.deletionPolicy` (`Delete`/`Orphan`), `spec.managementPolicies`, `spec.initProvider`,
`spec.writeConnectionSecretToRef` y `spec.providerConfigRef`.

Ejemplo mínimo:

```yaml
apiVersion: postgresql.postgresql.upbound.io/v1alpha1
kind: Database
metadata:
  name: mi-db
spec:
  forProvider:
    name: mi_db
  providerConfigRef:
    name: postgresql-config
```

---

## 3. Información requerida por el `ProviderConfig`

**apiVersion:** `postgresql.upbound.io/v1beta1` · **kind:** `ProviderConfig` (cluster-scoped, sin namespace).

El `ProviderConfig` no lleva los datos de conexión en el manifiesto: lleva un **puntero a dónde están**.

```yaml
apiVersion: postgresql.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: postgresql-config
spec:
  credentials:
    source: Secret            # case-sensitive: "Secret", no "secret"
    secretRef:
      namespace: crossplane-system
      name: postgresql-credentials
      key: connection         # la clave del Secret que contiene el JSON
```

| Campo | Qué indica |
|---|---|
| `spec.credentials.source` | De dónde salen las credenciales. Valores admitidos: `Secret`, `Environment`, `Filesystem`, `None`. Es **case-sensitive**. |
| `spec.credentials.secretRef.namespace` | Namespace del Secret (`crossplane-system`). |
| `spec.credentials.secretRef.name` | Nombre del Secret (`postgresql-credentials`). |
| `spec.credentials.secretRef.key` | **Clave dentro del Secret** cuyo valor es un **documento JSON** con los parámetros de conexión. En este laboratorio la clave se llama `connection`. |

### Contenido del JSON de credenciales

El provider hace `json.Unmarshal` del valor de esa clave a un `map[string]string` y de ahí toma los
parámetros de conexión (`internal/clients/postgresql.go`). Las claves reconocidas son:

| Clave JSON | Obligatoria | Descripción |
|---|---|---|
| `host` | **Sí** | Host del servidor PostgreSQL. En el clúster: `postgresql.postgresql.svc.cluster.local`. |
| `username` | **Sí** | Usuario de la conexión. Debe tener permisos para `CREATE DATABASE` (usamos `postgres`). |
| `password` | No (pero necesaria aquí) | Contraseña del usuario. |
| `port` | No — por defecto `5432` | Puerto. |
| `database` | No — por defecto `postgres` | Base de datos a la que se conecta el provider para ejecutar los comandos. |
| `sslmode` | No — por defecto `require` | Modo SSL. Dentro de Kind hay que poner `disable`. |
| `scheme` | No — por defecto `postgres` | Driver: `postgres`, `awspostgres`, `gcppostgres`. |
| `superuser` | No — por defecto `true` | Poner `"false"` si el usuario no es superusuario (RDS, Cloud SQL). |
| `sslrootcert` | No | Ruta al certificado raíz. |
| `connect_timeout` | No — `180` | Timeout de conexión en segundos. |
| `max_connections` | No — `20` | Máximo de conexiones abiertas. |
| `expected_version` | No — `9.0.0` | Versión esperada de PostgreSQL. |
| `database_username` | No | Usuario dentro de la base de datos si difiere del de la conexión. |
| `aws_rds_iam_auth`, `aws_rds_iam_profile`, `aws_rds_iam_region` | No | Autenticación IAM de AWS RDS. |
| `azure_identity_auth`, `azure_tenant_id` | No | Autenticación con identidad de Azure. |

> Importante **Todos los valores del JSON deben ser cadenas**, incluido el puerto: `"port":"5432"`, no `"port":5432`.
> Si no, el `json.Unmarshal` a `map[string]string` falla y el `ProviderConfig` no se puede usar.

Por eso el Secret se lo cree así:

```bash
kubectl create secret generic postgresql-credentials \
  --namespace crossplane-system \
  --from-literal=connection='{"host":"postgresql.postgresql.svc.cluster.local","port":"5432","username":"postgres","password":"platform123","database":"postgres","sslmode":"disable"}'
```
