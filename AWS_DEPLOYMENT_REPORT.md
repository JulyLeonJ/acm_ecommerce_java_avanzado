# Informe de preparación y despliegue AWS

- Fecha: 2026-10-22
- Región seleccionada: `us-east-2`
- Experiencia: nueva experiencia de AWS
- Resultado: se creó una conexión GitHub pendiente en CodeConnections y se adaptó el job de despliegue para CodeBuild Runner. No se crearon recursos de cómputo/red/base de datos: falta autorización GitHub y comprobar permisos IAM de CodeBuild.

## Trabajo en el repositorio

El commit `cd89472` contiene la preparación local previa:

- Gradle, CI y la imagen Docker alineados con Java 25.
- Configuración PostgreSQL mediante variables Spring, sin la contraseña fija que había en `application.yaml`.
- Actuator con `/actuator/health/readiness`, que incluye PostgreSQL.
- Workflow de GitHub Actions con publicación ECR por SHA y despliegue ECS, protegido por `ECS_DEPLOY_ENABLED=true`.
- `.dockerignore` y plantilla `ecs/task-definition.json` para Fargate.

En esta ejecución cambié el job de despliegue de OIDC a un CodeBuild-hosted GitHub Actions runner, eliminando `id-token: write` y `configure-aws-credentials`. También añadí este informe y `ECS_DEPLOYMENT.md`, con el diseño privado elegido, los prerrequisitos y las variables necesarias.

## Inventario consultado

Las consultas regionales se hicieron en `us-east-2` con el perfil local `acm-demo`.

| Recurso | Resultado |
| --- | --- |
| ECS clusters | Ninguno |
| Repositorios ECR | Ninguno |
| Instancias RDS | Ninguna |
| Load balancers | Ninguno |
| Secretos de la aplicación | Ninguno |
| CodeConnections | Se creó `acm-ecommerce-github` en `us-east-2`; estado actual `PENDING`, requiere autorizar GitHub App |
| VPC | Una VPC predeterminada |
| Subredes predeterminadas | Tres, en `us-east-2a`, `us-east-2b` y `us-east-2c`; asignan IP pública por defecto |
| Roles ECS/GitHub detectados por nombre | Solo el service-linked role de ECS; no se encontró un rol de despliegue ni execution role de la aplicación |

En GitHub confirmé que el workflow `CI and ECS deploy` está activo y que existen los secrets `DB_NAME`, `DB_USER` y `DB_PASSWORD` para CI. No hay variables de Actions configuradas; por tanto `ECS_DEPLOY_ENABLED` está sin establecer y el job de despliegue permanece apagado.

## Bloqueo de IAM/OIDC

Las operaciones `aws iam list-open-id-connect-providers` y `aws iam create-open-id-connect-provider` fueron rechazadas con `AccessDenied`: el SCP administrado del proyecto deniega explícitamente `iam:ListOpenIDConnectProviders` e `iam:CreateOpenIDConnectProvider`. Por eso no se usará GitHub OIDC.

No intenté eludir la política mediante access keys permanentes ni crear recursos que quedarían sin pipeline operativo. La llamada para actualizar el estado del plan y la consulta de grupos de logs también recibieron `429 Rate exceeded` del servicio de autenticación durante esta ejecución.

La última consulta de estado que había funcionado indicó plan `FREE` y USD 100 de créditos. No pude refrescarlo ahora por el error 429. RDS, Fargate, ALB, NAT/VPC endpoints, logs, ECR y tráfico pueden consumir créditos o generar cargos; revisa **AWS Settings > Billing**.

Para reanudar con OIDC, el propietario tendría que resolver con AWS el SCP. En vez de eso se prepara CodeBuild Runner, que entrega las credenciales desde un service role de CodeBuild y no requiere proveedor IAM OIDC.

## Alternativa evaluada

AWS CodeBuild ofrece **CodeBuild-hosted GitHub Actions runners**. Esta opción conserva el workflow GitHub Actions, ejecuta el job de despliegue en un runner efímero de CodeBuild y usa el service role de CodeBuild para AWS; no necesita crear un proveedor IAM OIDC. Requiere autorizar la GitHub App de la conexión pendiente, habilitar webhook `WORKFLOW_JOB_QUEUED`, modo privilegiado para Docker y un service role con permisos mínimos para ECR/ECS.

No creé todavía el CodeBuild project porque aún no se ha comprobado si el SCP permite `iam:CreateRole`, `iam:PassRole` y las operaciones de CodeBuild. La conexión GitHub `acm-ecommerce-github` quedó `PENDING`; requiere que la propietaria del repositorio autorice la AWS Connector GitHub App en Developer Tools > Settings > Connections. La alternativa CodePipeline + CodeConnections también necesita roles de servicio. La referencia y los pasos están en `ECS_DEPLOYMENT.md`.

## Validaciones

- `bash ./gradlew bootJar --no-daemon`: correcto.
- YAML del workflow y Spring, JSON de ECS, sintaxis `jq` y `git diff --check`: correctos.
- `bash ./gradlew test --no-daemon`: no pasó localmente porque `contextLoads()` requiere PostgreSQL en `localhost:5434` y no había una base disponible.
- Docker no estaba iniciado, así que no pude construir ni ejecutar la imagen localmente. El workflow de CI levanta PostgreSQL temporal si existen los secrets de CI `DB_NAME`, `DB_USER` y `DB_PASSWORD`.

## Seguridad y decisiones

- Se eligió mantener la API privada: el despliegue previsto usa ALB interno, sin ingress público.
- `DisableCsrf.java` permite todas las rutas sin autenticación y desactiva CSRF. No se debe exponer la API a Internet hasta implementar autenticación y autorización adecuadas.
- Al ser interno, el DNS del ALB no será accesible desde Internet ni desde la laptop sin conectividad privada a la VPC. La validación inicial puede hacerse con target health y logs; para peticiones desde la laptop se requiere VPN u otra ruta privada aprobada.
- La solicitud de conservar los recursos para validación queda anotada. Solo existe la conexión CodeConnections pendiente; no hay recursos de cómputo que generen cargos que conservar o limpiar.

## Pendiente

1. Autorizar AWS Connector for GitHub para el repositorio en la conexión `acm-ecommerce-github` y confirmar estado `AVAILABLE`.
2. Probar si SCP permite el service role de CodeBuild, `iam:PassRole` y el CodeBuild Runner project; detenerse si hay denegación explícita.
3. Crear RDS PostgreSQL no pública, secreto JSON, ECR, roles ECS, CloudWatch Logs y red privada.
4. Crear cluster/servicio Fargate y ALB interno; completar los ARN en `ecs/task-definition.json`.
5. Configurar `CODEBUILD_RUNNER_PROJECT`, `ECR_REPOSITORY`, `ECS_CLUSTER` y `ECS_SERVICE`; solo entonces habilitar `ECS_DEPLOY_ENABLED=true`.
6. Repetir `./gradlew test` con PostgreSQL disponible y validar el servicio interno.
