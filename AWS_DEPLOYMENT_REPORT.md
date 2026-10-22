# Informe de preparación y despliegue AWS

- Fecha: 2026-10-22
- Región seleccionada: `us-east-2`
- Experiencia: nueva experiencia de AWS
- Resultado: la infraestructura privada está creada y el servicio ECS espera en `desiredCount=0`. El despliegue sigue pendiente de publicar en GitHub los cambios locales del workflow.

## Trabajo en el repositorio

El commit `cd89472` contiene la preparación local previa:

- Gradle, CI y la imagen Docker alineados con Java 25.
- Configuración PostgreSQL mediante variables Spring, sin la contraseña fija que había en `application.yaml`.
- Actuator con `/actuator/health/readiness`, que incluye PostgreSQL.
- Workflow de GitHub Actions con publicación ECR por SHA y despliegue ECS, protegido por `ECS_DEPLOY_ENABLED=true`.
- `.dockerignore` y plantilla `ecs/task-definition.json` para Fargate.

En esta ejecución cambié localmente el job de despliegue de OIDC a un CodeBuild-hosted GitHub Actions runner, eliminando `id-token: write` y `configure-aws-credentials`. Actualicé la task definition con endpoint privado y secreto RDS, y añadí este informe y `ECS_DEPLOYMENT.md`. Los cambios del workflow/task definition aún no están committeados ni enviados a GitHub.

## Inventario consultado

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

En GitHub confirmé que el workflow `CI and ECS deploy` está activo y que existen los secrets `DB_NAME`, `DB_USER` y `DB_PASSWORD` para CI. Configuré `CODEBUILD_RUNNER_PROJECT`, `ECR_REPOSITORY`, `ECS_CLUSTER` y `ECS_SERVICE`; `ECS_DEPLOY_ENABLED=false` mantiene el job apagado.

## Recursos creados

| Recurso | Estado actual |
| --- | --- |
| ECR `acm-ecommerce` | Privado, tags inmutables, escaneo básico al subir y cifrado AES256; sin imagen todavía |
| CodeConnections `acm-ecommerce-github` | `AVAILABLE`, autorizado para el repositorio |
| CodeBuild Runner `acm-ecommerce-deploy-runner` | Webhook `WORKFLOW_JOB_QUEUED` activo, workflow filtrado, Docker privilegiado y concurrencia 1 |
| Roles IAM | Service role de CodeBuild y ECS task execution role creados con permisos específicos |
| Red privada | Dos subredes privadas en `us-east-2a`/`us-east-2b`, route table privada y un NAT Gateway en `us-east-2a` |
| RDS PostgreSQL | `acm-ecommerce-postgres`, 18.3, `db.t4g.micro`, Single-AZ, 20 GB gp3, cifrada y no pública; contraseña administrada por RDS/Secrets Manager |
| CloudWatch Logs | `/ecs/acm-ecommerce`, retención de 7 días |
| ECS | Cluster y servicio `acm-ecommerce`; task definition revisión 1; `desiredCount=0` |
| ALB | Interno, sin listener público; target group `ip` y readiness `/actuator/health/readiness` |

El CodeBuild Runner, ECR y RDS se crearon en `us-east-2`. El servicio no ejecuta tareas hasta que el workflow publique la primera imagen.

## Bloqueo de IAM/OIDC

Las operaciones `aws iam list-open-id-connect-providers` y `aws iam create-open-id-connect-provider` fueron rechazadas con `AccessDenied`: el SCP administrado del proyecto deniega explícitamente `iam:ListOpenIDConnectProviders` e `iam:CreateOpenIDConnectProvider`. Por eso no se usará GitHub OIDC.

No intenté eludir la política mediante access keys permanentes. Algunas consultas iniciales recibieron `429 Rate exceeded`, pero después pude confirmar el estado del plan y completar la configuración de los recursos.

La consulta actual indicó plan `FREE`, activo, con USD 100 de créditos hasta el 22 de abril de 2027. RDS, Fargate, ALB, NAT, logs, ECR y tráfico pueden consumir créditos o generar cargos. RDS, NAT y ALB ya están activos y facturan aunque ECS tenga cero tareas; revisa **AWS Settings > Billing**.

El SCP sigue bloqueando OIDC, pero el service role de CodeBuild se pudo crear. Se cambió la integración para usar CodeBuild Runner, que entrega credenciales desde ese service role sin proveedor IAM OIDC.

## Alternativa evaluada

AWS CodeBuild ofrece **CodeBuild-hosted GitHub Actions runners**. Esta opción conserva el workflow GitHub Actions, ejecuta el job de despliegue en un runner efímero de CodeBuild y usa el service role de CodeBuild para AWS; no necesita crear un proveedor IAM OIDC. La GitHub App quedó autorizada, y el proyecto tiene webhook `WORKFLOW_JOB_QUEUED`, modo privilegiado para Docker y un service role limitado para ECR/ECS.

El CodeBuild Runner ya está configurado. La variable `ECS_DEPLOY_ENABLED` se mantiene en `false`; el webhook no ejecutará el despliegue hasta que se publique el cambio local del workflow en GitHub y se habilite la variable.

## Validaciones

- `bash ./gradlew bootJar --no-daemon`: correcto.
- YAML del workflow y Spring, JSON de ECS, sintaxis `jq` y `git diff --check`: correctos.
- `bash ./gradlew test --no-daemon`: no pasó localmente porque `contextLoads()` requiere PostgreSQL en `localhost:5434` y no había una base disponible.
- Docker no estaba iniciado, así que no pude construir ni ejecutar la imagen localmente. El workflow de CI levanta PostgreSQL temporal si existen los secrets de CI `DB_NAME`, `DB_USER` y `DB_PASSWORD`.

## Seguridad y decisiones

- Se eligió mantener la API privada: el despliegue previsto usa ALB interno, sin ingress público.
- `DisableCsrf.java` permite todas las rutas sin autenticación y desactiva CSRF. No se debe exponer la API a Internet hasta implementar autenticación y autorización adecuadas.
- Al ser interno, el DNS del ALB no será accesible desde Internet ni desde la laptop sin conectividad privada a la VPC. La validación inicial puede hacerse con target health y logs; para peticiones desde la laptop se requiere VPN u otra ruta privada aprobada.
- La solicitud de conservar los recursos para validación queda anotada. El servicio ECS está en cero tareas, pero RDS, NAT y ALB quedan activos y pueden generar cargos mientras se espera el push.

## Pendiente

1. Obtener autorización explícita para commit/push del workflow y la task definition al branch `main`; esos cambios están en el worktree local.
2. Publicar el cambio y activar `ECS_DEPLOY_ENABLED=true` para iniciar el CodeBuild runner y publicar la primera imagen.
3. Confirmar que ECS inicia una tarea, que el target interno pasa readiness y que el log group recibe logs.
4. Repetir `./gradlew test` con PostgreSQL disponible y validar el servicio interno mediante conectividad privada.
