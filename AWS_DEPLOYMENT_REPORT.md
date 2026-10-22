# Informe de preparación y despliegue AWS

- Fecha: 2026-10-22
- Región seleccionada: `us-east-2`
- Experiencia: nueva experiencia de AWS
- Resultado: despliegue privado completado y validado. ECS mantiene una tarea saludable detrás de un ALB interno; la API no tiene ingress desde Internet.

## Trabajo en el repositorio

El commit base `cd89472` contenía la preparación local inicial. El commit `fbbc9a2` con el cambio a CodeBuild Runner, task definition y documentación está publicado en `main`.

- Gradle, CI y la imagen Docker alineados con Java 25.
- Configuración PostgreSQL mediante variables Spring, sin la contraseña fija que había en `application.yaml`.
- Actuator con `/actuator/health/readiness`, que incluye PostgreSQL.
- Workflow de GitHub Actions con publicación ECR por SHA y despliegue ECS, protegido por `ECS_DEPLOY_ENABLED=true`.
- `.dockerignore` y plantilla `ecs/task-definition.json` para Fargate.

En esta ejecución cambié el job de despliegue de OIDC a un CodeBuild-hosted GitHub Actions runner, eliminando `id-token: write` y `configure-aws-credentials`. Actualicé la task definition con el endpoint privado y el secreto RDS. Commit/push `fbbc9a2` publicó los cambios a `main`.

## Inventario inicial

Las consultas regionales se hicieron en `us-east-2` con el perfil local `acm-demo`.

| Recurso | Resultado |
| --- | --- |
| ECS clusters | Ninguno |
| Repositorios ECR | Ninguno |
| Instancias RDS | Ninguna |
| Load balancers | Ninguno |
| Secretos de la aplicación | Ninguno |
| CodeConnections iniciales | Ninguna |
| VPC | Una VPC predeterminada |
| Subredes predeterminadas | Tres, en `us-east-2a`, `us-east-2b` y `us-east-2c`; asignan IP pública por defecto |
| Roles ECS/GitHub detectados por nombre | Solo el service-linked role de ECS; no se encontró un rol de despliegue ni execution role de la aplicación |

En GitHub confirmé que el workflow `CI and ECS deploy` está activo y que existen los secrets `DB_NAME`, `DB_USER` y `DB_PASSWORD` para CI. Configuré `CODEBUILD_RUNNER_PROJECT`, `ECR_REPOSITORY`, `ECS_CLUSTER` y `ECS_SERVICE`; habilité `ECS_DEPLOY_ENABLED=true` para el despliegue.

## Recursos creados

| Recurso | Estado actual |
| --- | --- |
| ECR `acm-ecommerce` | Privado, tags inmutables, escaneo básico al subir y cifrado AES256; imagen `fbbc9a24a5ee2816a44474477d3670c41391d73a` (`sha256:3d3ce6fc6c6521887d1fe2649977bc68209f097bae9d5f2186b1770d036efbae`) |
| CodeConnections `acm-ecommerce-github` | `AVAILABLE`, autorizado para el repositorio |
| CodeBuild Runner `acm-ecommerce-deploy-runner` | Webhook `WORKFLOW_JOB_QUEUED` activo, workflow filtrado, Docker privilegiado y concurrencia 1 |
| Roles IAM | Service role de CodeBuild y ECS task execution role creados con permisos específicos |
| Red privada | Dos subredes privadas en `us-east-2a`/`us-east-2b`, route table privada y un NAT Gateway en `us-east-2a` |
| RDS PostgreSQL | `acm-ecommerce-postgres`, 18.3, `db.t4g.micro`, Single-AZ, 20 GB gp3, cifrada y no pública; contraseña administrada por RDS/Secrets Manager |
| CloudWatch Logs | `/ecs/acm-ecommerce`, retención de 7 días |
| ECS | Cluster y servicio `acm-ecommerce`; task definition revisión 2; `desiredCount=1`, `runningCount=1` |
| ALB | Interno, sin listener público; target group `ip`, task `172.31.79.149:8080` saludable en `/actuator/health/readiness` |

El CodeBuild Runner, ECR y RDS están en `us-east-2`. El endpoint del ALB es `internal-acm-ecommerce-internal-412399010.us-east-2.elb.amazonaws.com` y solo es accesible por red privada.

## Bloqueo de IAM/OIDC

Las operaciones `aws iam list-open-id-connect-providers` y `aws iam create-open-id-connect-provider` fueron rechazadas con `AccessDenied`: el SCP administrado del proyecto deniega explícitamente `iam:ListOpenIDConnectProviders` e `iam:CreateOpenIDConnectProvider`. Por eso no se usará GitHub OIDC.

No intenté eludir la política mediante access keys permanentes. Algunas consultas iniciales recibieron `429 Rate exceeded`, pero después pude confirmar el estado del plan y completar la configuración de los recursos.

La consulta más reciente indica plan `FREE`, activo, con USD 120 de créditos restantes y expiración el 22 de abril de 2027. RDS, Fargate, ALB, NAT, logs, ECR y tráfico consumen créditos o pueden generar cargos. RDS, NAT, ALB y una tarea Fargate están activos; revisa **AWS Settings > Billing**, ya que los cargos pueden reflejarse con retraso.

El SCP sigue bloqueando OIDC, pero el service role de CodeBuild se pudo crear. CodeBuild entrega credenciales temporales al runner sin proveedor IAM OIDC ni access keys AWS en GitHub.

## Alternativa evaluada

AWS CodeBuild ofrece **CodeBuild-hosted GitHub Actions runners**. Esta opción conserva el workflow GitHub Actions, ejecuta el job de despliegue en un runner efímero de CodeBuild y usa el service role de CodeBuild para AWS; no necesita crear un proveedor IAM OIDC. La GitHub App quedó autorizada, y el proyecto tiene webhook `WORKFLOW_JOB_QUEUED`, modo privilegiado para Docker y un service role limitado para ECR/ECS.

El CodeBuild Runner y su webhook están configurados. La variable `ECS_DEPLOY_ENABLED` está en `true`; el webhook ejecutó el job de despliegue.

## Validaciones

- `bash ./gradlew bootJar --no-daemon`: correcto.
- YAML del workflow y Spring, JSON de ECS, sintaxis `jq` y `git diff --check`: correctos.
- El workflow de GitHub Actions `39820672748` terminó `success`: `Start PostgreSQL`, `./gradlew clean build`, subida del artefacto, Docker build/push a ECR y actualización ECS. [Ver ejecución](https://github.com/JulyLeonJ/acm_ecommerce_java_avanzado/actions/runs/39820672748).
- El CodeBuild Runner `acm-ecommerce-deploy-runner` terminó `SUCCEEDED` (build `7e66a789-c45e-450f-b810-037020b5cded`).
- ECS `runningCount=1`; ALB reportó el target como `healthy`. Readiness incluye el health check de PostgreSQL.
- Docker no estaba iniciado localmente; la imagen se construyó en el CodeBuild Runner. La prueba completa sí pasó en GitHub Actions con PostgreSQL temporal.

## Seguridad y decisiones

- Se eligió mantener la API privada: el ALB es interno y no tiene ingress público.
- `DisableCsrf.java` permite todas las rutas sin autenticación y desactiva CSRF. No se debe exponer la API a Internet hasta implementar autenticación y autorización adecuadas.
- La aplicación usa por ahora el usuario master `acm_ecommerce_admin` de RDS para que Hibernate cree/actualice el esquema. Para producción, crea un usuario de aplicación con privilegios mínimos y usa migraciones versionadas en vez de `ddl-auto=update`.
- El DNS interno no es accesible desde Internet ni desde una laptop sin VPN/ruta privada. El target group saludable valida el endpoint de readiness; para llamar otras rutas desde la laptop se necesita conectividad privada.
- Los recursos quedan activos para que los valides, como pediste. RDS, NAT Gateway, ALB y Fargate generan cargos/créditos; revisa **AWS Settings > Billing**.

## Pendiente

1. Validar rutas de la API desde una conexión privada a la VPC.
2. Implementar autenticación/autorización de aplicación antes de permitir cualquier acceso público.
3. Después de tu validación, limpiar los recursos temporales si decides reducir costos.
