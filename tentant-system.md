# Sistema Tenant — Documentación de Arquitectura

## 1. Visión General (Nivel C4 — Contexto)

El sistema está construido sobre una arquitectura **multi-tenant**, donde cada empresa
cliente es un `tenant` independiente. Los usuarios pertenecen a un tenant y tienen
un rol que determina qué datos pueden ver y gestionar.
```mermaid
C4Context
    title Sistema Tenant — Contexto General

    Person(user, "Usuario / Persona", "Miembro de una empresa")
    Person(admin, "Company Admin", "Administrador de empresa")
    Person(superadmin, "Superadmin", "Administrador de plataforma")

    System(app, "Aplicación Móvil", "Genera telemetría y datos biométricos")
    System_Boundary(supabase, "Supabase / PostgreSQL") {
        System(api, "REST API", "Expone datos vía RLS por rol")
        System(db, "Base de Datos", "Almacena toda la información")
        System(auth, "Auth", "Gestiona identidad y roles")
    }

    Rel(user, app, "Usa")
    Rel(app, api, "Envía telemetría")
    Rel(admin, api, "Gestiona su tenant")
    Rel(superadmin, api, "Gestiona toda la plataforma")
    Rel(api, db, "Lee / Escribe")
    Rel(auth, api, "Valida tokens y roles")
```

---

## 2. Jerarquía Organizacional (Nivel C4 — Contenedor)

Cada tenant representa una empresa. Dentro de ella existe una jerarquía de
**departamento → equipo → célula → persona**.
```mermaid
C4Container
    title Jerarquía Organizacional del Tenant

    System_Boundary(tenant, "Tenant (Empresa)") {
        Container(t, "tenants", "Tabla", "Empresa cliente raíz del sistema")
        Container(cd, "company_departments", "Tabla", "Departamentos o gerencias de la empresa")
        Container(tm, "teams", "Tabla", "Equipos dentro de un departamento")
        Container(c, "cells", "Tabla", "Células dentro de un equipo")
        Container(mem, "tenant_memberships", "Tabla", "Vincula personas a la jerarquía completa")
    }

    Rel(t, cd, "tiene")
    Rel(cd, tm, "tiene")
    Rel(tm, c, "tiene")
    Rel(mem, t, "pertenece a")
    Rel(mem, cd, "asignado a")
    Rel(mem, tm, "asignado a")
    Rel(mem, c, "asignado a")
```

---

## 3. Tablas del Sistema Tenant

### 3.1 `tenants`
Tabla raíz. Cada fila representa una empresa cliente independiente.
Casi todas las tablas del sistema tienen `tenant_id` como foreign key hacia esta tabla.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | uuid | Identificador único del tenant |
| `name` | text | Nombre de la empresa |

**Políticas RLS:**
- Solo miembros del tenant pueden ver su propio tenant
- `service_role` tiene acceso total

---

### 3.2 `company_departments`
Representa las gerencias o departamentos dentro de una empresa.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | uuid | Identificador único |
| `tenant_id` | uuid | FK → tenants |
| `name` | text | Nombre del departamento |

**Políticas RLS:**
- Cualquier miembro del tenant puede **ver** sus departamentos
- Solo `company_admin` y `superadmin` pueden **gestionar**

---

### 3.3 `teams`
Equipos dentro de un departamento.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | uuid | Identificador único |
| `tenant_id` | uuid | FK → tenants |
| `department_id` | uuid | FK → company_departments |

**Políticas RLS:**
- Cualquier miembro del tenant puede **ver** los equipos
- Solo `company_admin` y `superadmin` pueden **gestionar**

---

### 3.4 `cells`
Subdivisiones dentro de un equipo.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | uuid | Identificador único |
| `team_id` | uuid | FK → teams |

**Políticas RLS:**
- Cualquier miembro del tenant puede **ver** las células
- Solo `company_admin` y `superadmin` pueden **gestionar**

---

### 3.5 `tenant_memberships`
Tabla pivote clave. Vincula a un usuario con todos los niveles de la jerarquía
simultáneamente. Es el punto de entrada para resolver a qué empresa, departamento,
equipo y célula pertenece una persona.

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `id` | uuid | Identificador único |
| `tenant_id` | uuid | FK → tenants |
| `department_id` | uuid | FK → company_departments |
| `team_id` | uuid | FK → teams |
| `cell_id` | uuid | FK → cells |
| `user_id` | uuid | FK → auth.users |
| `role` | app_role | Rol del usuario en este tenant |

**Políticas RLS:**
- El usuario ve solo su propia membresía
- `company_admin` ve todas las membresías de su tenant
- `superadmin` ve todo

---

## 4. Roles del Sistema
```mermaid
graph TD
    SA[superadmin\nVe y gestiona toda la plataforma]
    CA[company_admin\nGestiona su tenant completo]
    TA[team_admin\nGestiona su equipo]
    CO[coach\nVe datos de salud de su equipo]
    US[user\nVe solo sus propios datos]
    SR[service_role\nAcceso total - solo backend]

    SA --> CA --> TA --> CO --> US
    SR -. bypass RLS .-> SA
```

| Rol | Alcance | Puede gestionar |
|-----|---------|-----------------|
| `superadmin` | Toda la plataforma | Todo |
| `company_admin` | Su tenant | Departamentos, equipos, células, miembros |
| `team_admin` | Su equipo | Miembros del equipo |
| `coach` | Su equipo | Solo lectura de datos de salud |
| `user` | Solo sus datos | Sus propios registros |
| `service_role` | Todo (bypass RLS) | Procesos backend / ETL |

---

## 5. Flujo de Datos de Telemetría
```mermaid
flowchart LR
    APP(App Móvil) -->|telemetría| BIO(biometrics_timeseries)
    APP -->|métricas| MR(metric_records)
    APP -->|registros médicos| MED(medical_records)

    BIO --> DEV(devices)
    MR --> MD(metric_definitions)
    MED --> OBS(observations)
    MED --> MF(medical_files)

    BIO & MR & MED -->|tenant_id| T(tenants)
```

---

## 6. Notas y Deuda Técnica

> ⚠️ **Sistema dual:** Existen dos jerarquías paralelas en la base de datos.
> El sistema `ios_teams / organizations` (legacy) y el sistema `tenants / company_departments`
> (principal). Antes de escalar la reportería conviene confirmar si se van a unificar.

> ⚠️ **Reports con `user_id NULL`:** Cualquier miembro del tenant puede ver reportes
> sin dueño asignado. Revisar si esto es intencional.

> ✅ **`service_role` para ETL:** El pipeline Bronze → Silver → Gold debe usar
> exclusivamente la `service_role key` para bypasear RLS y acceder a todos los datos.