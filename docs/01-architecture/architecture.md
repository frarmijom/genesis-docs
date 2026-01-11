# Genesis – Arquitectura de Plataforma

## 1. Contexto

Genesis es una plataforma backend diseñada para ejecutar cálculos matemáticos de alta carga computacional (Monte Carlo) de forma confiable, trazable y reproducible, con persistencia de resultados en base de datos y operación continua en ambientes productivos.

El sistema fue construido con los siguientes objetivos arquitectónicos:

- Separar estrictamente los ambientes de **construcción, validación y producción**
- Permitir **despliegue continuo controlado (CI/CD)**
- Asegurar **trazabilidad, observabilidad y reproducibilidad**
- Ejecutar cálculos intensivos de CPU sin afectar estabilidad
- Permitir rollback, troubleshooting y operación 24/7

---

## 2. Ambientes

La plataforma se ejecuta en tres máquinas virtuales independientes:

| Entorno | Hostname | IP |
|--------|----------|----|
| BUILD  | prd06    | 192.168.1.207 |
| TEST   | prd07    | 192.168.1.208 |
| PROD   | prd08    | 192.168.1.209 |

Cada entorno tiene un rol bien definido:

| Entorno | Rol |
|--------|-----|
| prd06 | Compilación, tests, empaquetado y orquestación CI/CD |
| prd07 | Validación funcional del release |
| prd08 | Servicio productivo expuesto a clientes |

---

## 3. Arquitectura de Alto Nivel

La arquitectura sigue el patrón **CI/CD + ambientes segregados + promoción controlada**.

### Componentes principales

Cada entorno contiene:

- Jenkins Agent
- Servicio Genesis (Spring Boot)
- PostgreSQL (Docker)

Jenkins en `prd06` coordina:

- Checkout desde GitHub
- Build y tests
- Promoción a TEST
- Smoke tests
- Aprobación humana
- Promoción a PROD

Los usuarios finales solo interactúan con `prd08`, vía API HTTP/JSON.

---

## 4. Flujo CI/CD

El pipeline sigue este flujo:

1. El desarrollador hace `git push` a GitHub  
2. Jenkins (`prd06`) clona el repo  
3. Ejecuta:
   - Compilación
   - Tests unitarios
   - Empaquetado del JAR  
4. El artefacto es desplegado en `prd07`  
5. Se ejecutan pruebas de humo (health + POST)  
6. Un operador aprueba el despliegue  
7. El mismo artefacto se despliega en `prd08`  
8. Se valida disponibilidad productiva  

Esto asegura:

- **Lo que se prueba es exactamente lo que se produce**
- No hay builds en PROD
- No hay cambios manuales en producción

---

## 5. Arquitectura interna de la aplicación

Genesis es una aplicación **Spring Boot** con arquitectura en capas:


HTTP API
↓
Controller (REST)
↓
Service Layer (Monte Carlo Engine)
↓
JPA / Hibernate
↓
PostgreSQL



### Flujo de una petición

1. Un cliente envía un POST a `/api/v1/calculations`  
2. El Controller valida la request  
3. El Service ejecuta el cálculo Monte Carlo  
4. Se mide:
   - tiempo de CPU  
   - duración  
   - error estadístico  
5. El resultado se guarda en PostgreSQL  
6. Se devuelve el resultado al cliente  

---

## 6. Modelo de ejecución

El servicio se ejecuta como:

Usuario: genesis
Proceso: java -jar genesis.jar
Control: systemd



Systemd gestiona:

- Arranque
- Reinicio automático
- Logging
- Health del proceso

Esto permite:

- Reinicios controlados
- Integración con monitoreo
- Operación sin intervención humana

---

## 7. Seguridad

- Jenkins usa SSH con llaves
- Los servicios corren con usuario no-root (`genesis`)
- Los archivos de configuración tienen permisos 640
- La base de datos corre aislada en Docker
- No hay credenciales hardcodeadas en el JAR

---

## 8. Beneficios de esta arquitectura

| Beneficio | Cómo se logra |
|--------|---------------|
| Confiabilidad | Systemd + health checks |
| Reproducibilidad | Artefactos inmutables |
| Seguridad | Usuarios dedicados + SSH |
| Escalabilidad | App desacoplada de DB |
| Observabilidad | Actuator + logs |
| Gobernanza | Approval manual a PROD |
