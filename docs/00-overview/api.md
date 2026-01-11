# Genesis API – Especificación (v1)

Base URL por ambiente:

- TEST: `http://prd07:8080`
- PROD: `http://prd08:8080`

Headers comunes:
- `Content-Type: application/json`

---

## Health

### GET `/actuator/health`
Verifica que la aplicación está viva y lista.

**Respuesta (200)**
```json
{"status":"UP"}

Cálculos
POST /api/v1/calculations

Ejecuta un cálculo Monte Carlo, persiste el resultado y devuelve el registro.

Request body


{
  "functionType": "GAUSSIAN_SIN",
  "a": -2,
  "b": 2,
  "samplesN": 200000,
  "seed": 42
}

Campos

functionType (string/enum): tipo de función a evaluar.

a (number): límite inferior.

b (number): límite superior.

samplesN (integer): número de muestras Monte Carlo.

seed (integer, opcional): semilla para reproducibilidad.

Respuestas

201 o 200: cálculo creado/ejecutado exitosamente.

400: validación fallida (ej: a >= b, samplesN <= 0, tipo inválido).

500: error interno (ej: DB caída, excepción de cálculo).

Response body (ejemplo)

{
  "id": 123,
  "functionType": "GAUSSIAN_SIN",
  "a": -2.0,
  "b": 2.0,
  "samplesN": 200000,
  "seed": 42,
  "result": 1.2345,
  "stdError": 0.0021,
  "ciLow": 1.2304,
  "ciHigh": 1.2386,
  "durationMs": 430,
  "cpuTimeMs": 410,
  "status": "DONE",
  "disabled": false,
  "createdAt": "2026-01-09T15:00:00Z",
  "updatedAt": "2026-01-09T15:00:00Z"
}

GET /api/v1/calculations/{id} (consulta)


sequenceDiagram
  autonumber
  participant C as Cliente
  participant API as Controller
  participant SVC as CalculationService
  participant DB as PostgreSQL (JPA)

  C->>API: GET /api/v1/calculations/{id}
  API->>SVC: findById(id)
  SVC->>DB: findById(id)
  DB-->>SVC: Entity / empty
  SVC-->>API: Response / NotFound
  API-->>C: 200 (JSON) / 404


sequenceDiagram
  autonumber
  participant C as Cliente
  participant API as Controller
  participant SVC as CalculationService
  participant DB as PostgreSQL (JPA)

  C->>API: DELETE /api/v1/calculations/{id}
  API->>SVC: disable(id)
  SVC->>DB: findById(id)
  DB-->>SVC: Entity
  SVC->>SVC: entity.disabled=true
  SVC->>DB: save(entity)
  DB-->>SVC: OK
  SVC-->>API: OK
  API-->>C: 204 No Content



---

## 3) `docs/05-ops-runbooks/runbook-production.md` (Runbook de producción)

```md
# Runbook – Operación en Producción (prd08)

Este documento define procedimientos operativos para incidentes comunes de Genesis en PROD.

---

## 1. Datos del entorno

- PROD: `prd08` (`192.168.1.209`)
- Servicio: `genesis.service`
- Usuario del servicio: `genesis`
- JAR: `/opt/genesis/app/genesis.jar`
- Config: `/opt/genesis/config/application-prod.yml`
- Puerto: `8080`
- Health: `http://localhost:8080/actuator/health`

---

## 2. Chequeos rápidos

### 2.1 Ver si la app responde
```bash
curl -fsS http://localhost:8080/actuator/health | cat
echo


sudo systemctl status genesis --no-pager -l

sudo journalctl -u genesis -n 120 --no-pager

sudo ss -lptn | grep ':8080' || echo "NO LISTEN 8080"


sudo systemctl restart genesis
sudo systemctl is-active --quiet genesis && echo "Service active" || echo "Service NOT active"


for i in $(seq 1 90); do
  if curl -fsS http://localhost:8080/actuator/health >/dev/null 2>&1; then
    echo "Genesis UP"
    curl -fsS http://localhost:8080/actuator/health | cat
    echo
    exit 0
  fi
  sleep 1
done

echo "Genesis no levantó en 90s"
sudo systemctl status genesis --no-pager -l || true
sudo journalctl -u genesis -n 200 --no-pager || true
exit 1
ls -lah /opt/genesis/config/application-prod.yml
# esperado: -rw-r----- genesis genesis
