# Laboratorio: DevSecOps con Jenkins y Nexus

Este repositorio contiene la base para el laboratorio práctico de Seguridad Informática.

## Contenido del Proyecto

- **App Node.js**: Aplicación simple con lógica de suma y tests unitarios.
- **Infraestructura**: `docker-compose.yml` para levantar Jenkins y Nexus.
- **Pipeline**: `Jenkinsfile` base e incompleto.
- **Seguridad**: `Jenkinsfile.seg` con pistas para controles DevSecOps.


## Advertencia Técnica

Este proyecto **no está listo para funcionar mediante un "click"**.
Parte del ejercicio consiste en:
- Analizar por qué el `Jenkinsfile` falla.
- Corregir el networking entre contenedores (pista: no usen `localhost` dentro de Jenkins).
- Configurar credenciales de forma segura.

## Inicio Rápido

1. Crear un repositorio propio en GitHub y subir este código.
2. Levantar la infraestructura local:
   ```bash
   docker compose up -d
   docker compose logs -f
   ```
3. Acceder a los servicios:
   - Jenkins: [http://localhost:8080](http://localhost:8080) (`admin` / `admin`)
   - Nexus: [http://localhost:8081](http://localhost:8081)

## Credenciales Nexus

Para obtener la contraseña inicial de Nexus, ejecute:

```bash
docker exec -it <nombre_contenedor_nexus> cat /nexus-data/admin.password ; echo
```

