# Genesis – Aplicación (Arquitectura, Diseño y Lógica)

Este documento describe en detalle la aplicación **Genesis**: su arquitectura interna, diseño por capas, responsabilidades de componentes, flujo de ejecución y persistencia.

---

## 1. Propósito funcional

Genesis expone una API REST para ejecutar cálculos matemáticos de complejidad media/alta en CPU (modelo Monte Carlo), almacenar los resultados y permitir operaciones sobre esos registros.

Características principales:

- Ejecución de cálculos intensivos en CPU con parámetros de entrada.
- Persistencia de resultados y metadatos.
- API para crear, consultar, actualizar y deshabilitar registros (soft delete).
- Medición básica de performance (duración y tiempo de CPU).
- Endpoint de salud vía Spring Boot Actuator.

---

## 2. Stack tecnológico

- Java + Spring Boot
- Spring Web (REST)
- Spring Data JPA (Hibernate)
- PostgreSQL
- Actuator (health)
- Lombok (para reducir boilerplate, según proyecto)

---

## 3. Arquitectura interna por capas

Genesis implementa una arquitectura clásica en capas:

Cliente HTTP
↓
Controller (REST)
↓
Service (orquestación y reglas)
↓
Repositorio JPA (persistencia)
↓
PostgreSQL


### Objetivo de la separación
- **Controller**: HTTP + validación de entrada + mapping DTO.
- **Service**: reglas de negocio, ejecución Monte Carlo, persistencia, estados.
- **Repository**: abstracción de acceso a datos.
- **Entity**: modelo persistente.
- **DTO**: contratos de entrada/salida de la API.

---

## 4. Estructura de paquetes

Estructura típica (según el proyecto):


cl.armijo.genesis
├─ GenesisApplication.java
├─ controller/
│ └─ CalculationController.java
├─ service/
│ └─ CalculationService.java
├─ repository/
│ └─ CalculationRepository.java
├─ model/
│ ├─ dto/
│ │ ├─ CalculationCreateRequest.java
│ │ └─ CalculationResponse.java
│ └─ entity/
│ └─ Calculation.java
└─ math/
└─ MonteCarloIntegrator.java



---

## 5. Modelo de dominio (Entity)

### Entidad: `Calculation`

Representa un cálculo ejecutado, incluyendo entrada, salida y metadatos.

Campos típicos:

- `id`: identificador único.
- `functionType`: tipo de función a integrar/simular (ej: `GAUSSIAN_SIN`).
- Parámetros de entrada:
  - `a`, `b`: límites (intervalo).
  - `samplesN`: número de muestras Monte Carlo.
  - `seed`: semilla aleatoria (reproducibilidad).
- Resultado estadístico:
  - `result`: valor estimado.
  - `stdError`: error estándar estimado.
  - `ciLow`, `ciHigh`: intervalo de confianza.
- Telemetría:
  - `durationMs`: tiempo total del cálculo.
  - `cpuTimeMs`: tiempo de CPU medido (si está habilitado).
- Estado:
  - `status`: estado del cálculo (ej: `DONE`, `FAILED`, etc.).
  - `disabled`: soft delete (deshabilitado).
- Auditoría:
  - `createdAt`, `updatedAt`

> Nota: “disabled” permite conservar el historial sin borrar físicamente.

---

## 6. Contratos API (DTOs)

### Request: `CalculationCreateRequest`

Entrada esperada para ejecutar un cálculo:

- `functionType`: enum/string que identifica la función a simular.
- `a`, `b`: límites del intervalo.
- `samplesN`: cantidad de muestras Monte Carlo.
- `seed`: semilla opcional/recomendada.

Validaciones típicas:

- `samplesN > 0`
- `a < b`
- `functionType` no nulo

### Response: `CalculationResponse`

Representa la salida al cliente:

- Identidad del cálculo: `id`
- Parámetros de entrada reflejados
- Resultado (`result`, error estándar, intervalos)
- Métricas de performance
- Estado del registro (`disabled`, `status`)
- Timestamp de creación

---

## 7. Lógica matemática (Monte Carlo)

### Propósito
El núcleo del cálculo es una integración/estimación estadística mediante Monte Carlo.

Ejemplo conceptual:
- Elegir `x` aleatorios uniformemente en `[a,b]`
- Evaluar una función `f(x)`
- Estimar integral:
  - `I ≈ (b-a) * average(f(x))`

### Reproducibilidad
El parámetro `seed` permite reproducir resultados:
- Si `seed` es fijo, la secuencia pseudoaleatoria es determinística.
- Si no se provee, el sistema puede generar una semilla.

### Telemetría
Durante el cálculo se miden:
- `durationMs`: (wall-clock) tiempo real desde inicio a fin.
- `cpuTimeMs`: tiempo de CPU del thread (si se usa `ThreadMXBean`).

---

## 8. Service Layer (orquestación)

### `CalculationService`

Responsabilidades:

1. Validar reglas de negocio adicionales.
2. Ejecutar el integrador Monte Carlo con los parámetros.
3. Construir el objeto `Calculation` persistente.
4. Persistir a través de `CalculationRepository`.
5. Transformar Entity → Response DTO.

### Estados y errores
- Si el cálculo se ejecuta correctamente: `status=COMPLETED` (o `DONE`).
- Si falla por excepción (ej: argumentos inválidos, overflow, DB caída):
  - se registra `status=FAILED`
  - se escribe log de error
  - se retorna código HTTP apropiado desde el controller (idealmente 400 o 500 según corresponda).

---

## 9. Persistencia (Repository)

### `CalculationRepository`

Basado en Spring Data JPA:

- `save(...)`
- `findById(...)`
- `findAll(...)` (idealmente paginado)
- queries específicas (por ejemplo, excluir `disabled=true`)

Recomendación de diseño:
- Exponer endpoints que consulten por `disabled=false` por defecto.
- Mantener endpoint de auditoría (solo admin) si se quiere ver `disabled=true`.

---

## 10. API REST (Controller)

### `CalculationController`

Responsabilidades:

- Definir rutas REST.
- Recibir `CalculationCreateRequest` y validarlo (Bean Validation).
- Invocar `CalculationService`.
- Mapear respuestas y códigos HTTP.

Rutas típicas (referenciales):

- `POST /api/v1/calculations` → ejecuta un cálculo y lo persiste.
- `GET /api/v1/calculations/{id}` → consulta por id.
- `GET /api/v1/calculations` → lista (ideal paginación).
- `PUT/PATCH /api/v1/calculations/{id}` → actualiza campos permitidos.
- `DELETE /api/v1/calculations/{id}` → soft delete (set `disabled=true`).

---

## 11. Observabilidad y Health

Genesis expone Actuator:

- `GET /actuator/health`

Esto permite:

- Smoke tests en Jenkins
- Integración con monitoreo externo
- Validar disponibilidad en despliegues

Logging:

- La aplicación escribe logs a `stdout/stderr` (capturado por `systemd/journald`).
- Estos logs pueden ser enviados a Splunk Forwarder (pendiente/por implementar según runbook).

---

## 12. Ejemplo de flujo completo (POST)

1. Cliente envía:
   - `functionType`, `a`, `b`, `samplesN`, `seed`.
2. Controller valida el request.
3. Service ejecuta el cálculo Monte Carlo.
4. Service construye `Calculation` con:
   - input + output + métricas.
5. Repository guarda el registro en PostgreSQL.
6. Se retorna `CalculationResponse`.

---

## 13. Consideraciones de rendimiento

Monte Carlo escala linealmente con `samplesN`:

- Más muestras → mayor precisión estadística (menor error)
- Más muestras → más tiempo de CPU

Controles recomendados:

- límites máximos para `samplesN` (para evitar abuso)
- timeouts operacionales
- protección ante múltiples requests paralelos (si el host tiene CPU limitada)

---

## 14. Consideraciones de seguridad

- Configuración sensible fuera del repo, en:
  - `/opt/genesis/config/application-prod.yml` (permisos `640 genesis:genesis`)
- Proceso corre como usuario no-root (`genesis`)
- Jenkins despliega sin leer configuración sensible

---

## 15. Roadmap de mejoras (opcional)

- Métricas Prometheus (`/actuator/prometheus`)
- Paginación real en listados
- Endpoint de búsqueda por filtros (fecha, status, functionType)
- Rate limit / throttling
- Export de resultados a CSV
- Circuit breaker para DB (si se externaliza)
- Reporte de performance por request
