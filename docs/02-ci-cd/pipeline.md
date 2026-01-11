# Jenkins Pipeline – Genesis

El pipeline ejecuta las siguientes etapas:

1. Checkout (LinuxBuild / prd06)
2. Build + Unit Tests
3. Package
4. Deploy to TEST
5. Smoke TEST
6. Approval
7. Deploy to PROD
8. Smoke PROD

El despliegue es vía SSH usando el usuario `jenkins_node`.
