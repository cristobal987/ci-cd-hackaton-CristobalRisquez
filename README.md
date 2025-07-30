# ci-cd-hackaton-CristobalRisquez
proyecto para el hackathon

# categoria
basico

## CI/CD Pipeline

Este proyecto incluye un pipeline de CI/CD automatizado que se ejecuta en GitHub Actions y despliega en AWS.

### Características del Pipeline:

- **Continuous Integration (CI):**
  - Ejecuta tests automáticamente en cada push/PR
  - Verifica la calidad del código
  - Construye la aplicación Docker

- **Continuous Deployment (CD):**
  - Despliega automáticamente a AWS en commits a la rama main/master
  - Utiliza Amazon ECR para almacenar imágenes Docker
  - Despliega en Amazon ECS con Fargate

### Archivos de configuración:

- `.github/workflows/deploy.yml` - Pipeline de CI/CD
- `Dockerfile` - Configuración del contenedor
- `task-definition.json` - Definición de tarea de ECS
- `healthcheck.js` - Health check para el contenedor
- `AWS_SETUP.md` - Instrucciones de configuración de AWS

### Para desarrollo local:

```bash
# Instalar dependencias
npm install

# Ejecutar en modo desarrollo
npm run dev

# Ejecutar en modo producción
npm start
```

### Para probar con Docker:

```bash
# Construir imagen
docker build -t ci-cd-hackathon-app .

# Ejecutar contenedor
docker run -p 3000:3000 ci-cd-hackathon-app
```

La aplicación estará disponible en http://localhost:3000