# Genesis – Overview

Genesis es una aplicación Spring Boot que ejecuta cálculos matemáticos de alta carga de CPU (Monte Carlo) y persiste resultados en PostgreSQL.

El sistema fue diseñado para operar en tres entornos:
- BUILD (prd06)
- TEST (prd07)
- PROD (prd08)

El despliegue se realiza vía Jenkins Pipeline con promoción controlada.
