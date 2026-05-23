# 🐶 Solución DevOps: Tienda de Perritos - Innovatech Chile
## Documentación Técnica de Contenedorización, Gestión de Imágenes y Automatización CI/CD
### Asignatura: Introducción a Herramientas DevOps (ISY1101) | Duoc UC

---

## 1. Fundamentos Técnicos del Encargo
El presente documento detalla la implementación técnica para la migración, empaquetamiento y automatización del despliegue cloud de la aplicación multicapa **Tienda de Perritos**. La solución abarca desde el diseño de los entornos aislados mediante Docker hasta la orquestación de entrega continua utilizando GitHub Actions y servicios nativos de Amazon Web Services (AWS).

---

## 2. Contenedorización del Frontend y Backend (IE1)
Para cumplir con los estándares de rendimiento, portabilidad y seguridad requeridos, se diseñó e implementó una estrategia de empaquetamiento por capas utilizando Dockerfiles personalizados en cada microservicio:

* **Estructura Multi-stage Build:** Se configuraron compilaciones en múltiples etapas. La fase inicial se encarga de resolver las dependencias de desarrollo y construir los binarios/artefactos, mientras que la etapa final copia estrictamente los elementos ejecutables a una imagen base minimalista de producción, reduciendo el peso de la imagen y la superficie de ataque.
* **Principio de Mínimo Privilegio:** Se configuraron usuarios específicos no-root dentro de los Dockerfiles para evitar que los procesos internos de Node.js o el servidor web corran con privilegios de administrador del sistema.
* **Optimización y Limpieza:** Se estructuraron los comandos `RUN` minimizando el número de capas intermedias y eliminando los cachés de paquetes inmediatamente después de la instalación.

---

## 3. Orquestación Local con Docker Compose (IE2)
En el directorio raíz del proyecto se consolidó el archivo `docker-compose.yml` que empaqueta y define el stack completo de la aplicación para entornos de desarrollo local y pruebas de integración:

* **Servicios Definidos:**
  * `tienda-frontend`: Mapea el tráfico web exponiendo el puerto externo `80`.
  * `tienda-backend`: Procesa la lógica de negocio escuchando internamente en el puerto `3001`.
  * `tienda-db`: Levanta el motor relacional basado en una imagen de MySQL Server.
* **Políticas de Red y Dependencias:** Se configuraron redes internas cerradas (`networks`) para asegurar que el motor de base de datos no tenga exposición externa y se parametrizó la directiva `depends_on` para orquestar el orden correcto de inicialización de los servicios.

---

## 4. Estrategia de Persistencia de Datos (IE3)
Para asegurar la continuidad operativa y evitar la pérdida de registros de la Tienda de Perritos durante los reinicios de los contenedores o actualizaciones del pipeline, se implementó persistencia en la base de datos:

* **Mecanismo Utilizado:** *Named Volume* (Volumen Nombrado) denominado `dbdata`.
* **Ruta del Mapeo:** Vinculado internamente a la ruta de almacenamiento por defecto del motor relacional: `/var/lib/mysql`.
* **Justificación DevOps:** A diferencia de un *bind mount* (que amarra el contenedor a una ruta rígida y local del host), el *named volume* es gestionado directamente por el ciclo de vida de Docker. Esto garantiza que los datos persistan independientemente de que el contenedor sea destruído, recreado o actualizado por una nueva imagen del pipeline, delegando la persistencia de forma aislada en el almacenamiento del sistema operativo de la instancia virtual.

---

## 5. Gestión, Versionado y Registro de Imágenes (IE4)
De acuerdo a las directrices de la Actividad Práctica 2.4, la gestión y el ciclo de vida de las imágenes del proyecto se distribuyeron bajo un modelo híbrido:

* **Docker Hub (Registro Público):** Se utilizó para alojar la línea base inicial y el versionado semántico explícito (`usuario_Docker_hub/tienda-backend:v1`). Su rol principal es asegurar la **portabilidad**, permitiendo que cualquier desarrollador clone el proyecto y descargue las imágenes localmente mediante `docker login` y `docker push/pull` de forma universal.
* **Amazon ECR (Registro Privado):** Se configuraron repositorios privados nativos en AWS Elastic Container Registry (`tienda-frontend`, `tienda-backend`, `tienda-db`) utilizando el tag `:latest`. ECR actúa como el almacén exclusivo para el entorno productivo, garantizando gobernanza de seguridad mediante roles IAM y una transferencia de red de alta velocidad hacia los servidores.

---
## 6. Automatización del Pipeline CI/CD con GitHub Actions (IE4)
Siguiendo los flujos de la Actividad Práctica 2.5, se estructuraron workflows independientes (`.github/workflows/`) en formato YAML, configurados para dispararse ante eventos de `push` condicionados por rutas específicas (paths) de las carpetas:

[Código Fuente VS Code] ──> git push ──> [GitHub Actions] ──> [Amazon ECR] ──> [AWS SSM] ──> [EC2 Destinada]

---
### Flujo de Ejecución Técnico del Pipeline:
1. **Fase de Build y Autenticación:** Al gatillarse el workflow, el runner virtual de GitHub (`ubuntu-latest`) utiliza los **GitHub Secrets** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) para validar permisos con AWS Academy y compilar la nueva imagen Docker desde el código fuente modificado.
2. **Fase de Push:** La imagen pasa por el proceso de `docker tag` y se inyecta de forma segura mediante un `docker push` hacia el registro privado de Amazon ECR.
3. **Fase de Deploy (Orquestación Segura):** El pipeline invoca de forma remota la herramienta **AWS Systems Manager (SSM) Send-Command** apuntando al ID específico de la instancia (`EC2_FRONTEND_INSTANCE_ID`, etc.). El agente SSM interno de la máquina recibe la instrucción de manera encriptada y ejecuta de forma local los comandos en caliente: se autentica en ECR, realiza un `docker pull`, detiene y elimina el contenedor obsoleto (`docker stop` / `docker rm`) y levanta el servicio actualizado (`docker run`) logrando un despliegue con **Cero Tiempo de Inactividad (Zero Downtime)**.

---

## 7. Infraestructura Cloud, Funcionamiento e Integración (IE5, IE6, IE7)
La arquitectura productiva en la nube se encuentra desplegada de manera distribuida aprovechando **tres instancias de AWS EC2 independientes**, encapsuladas dentro de una VPC de laboratorio y aisladas lógicamente mediante **Security Groups** dedicados:

* **Capa Web - Instancia Frontend (`EC2-WEB`):** Aloja el contenedor del Frontend escuchando en el puerto `80`. Su Grupo de Seguridad (`SG-Front`) cuenta con reglas de entrada públicas (`0.0.0.0/0`) en el puerto HTTP 80. Es la única ventana de comunicación expuesta a Internet, validando el requerimiento del negocio.
* **Capa de Aplicación - Instancia Backend (`EC2-APP`):** Ejecuta la API del negocio en el puerto `3001`. Su Grupo de Seguridad (`SG-Back`) está blindado: solo acepta tráfico entrante cuyo origen coincida de forma estricta con la IP pública/privada de la máquina de Frontend.
* **Capa de Datos - Instancia Base de Datos (`EC2-DB`):** Contiene el motor relacional en el puerto `3306`. Su acceso de red está completamente cerrado al exterior, permitiendo peticiones única y exclusivamente desde la IP privada de la máquina de Backend.

Esta división física y lógica garantiza que, aunque un actor externo interactúe con la interfaz de la tienda en el navegador web, jamás podrá tener acceso o visibilidad directa hacia el backend o las tablas de datos relacionales, cumpliendo a cabalidad con las políticas de seguridad informática exigidas por Tienda Perritos.

