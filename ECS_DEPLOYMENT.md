# Despliegue privado en ECS/Fargate

La región del proyecto es `us-east-2`. La API se mantendrá privada, como se acordó. El workflow de `.github/workflows/build-java.yml` ejecuta CI, pero el job de despliegue solo se ejecutará en `main` cuando `ECS_DEPLOY_ENABLED=true`.

## Recursos necesarios

1. Crea un repositorio privado ECR llamado `acm-ecommerce` en `us-east-2`.
2. Autoriza la conexión CodeConnections `acm-ecommerce-github` que ya quedó creada en estado `PENDING`. En la consola de AWS, abre **Developer Tools > Settings > Connections**, selecciona esa conexión, elige **Update pending connection**, instala/autoriza **AWS Connector for GitHub** para `JulyLeonJ/acm_ecommerce_java_avanzado` y termina con **Connect**. Hazlo con la sesión de GitHub propietaria del repositorio.
3. Crea un CodeBuild **Runner project** llamado `acm-ecommerce-deploy-runner`, vinculado a ese repositorio/conexión y configurado para eventos `WORKFLOW_JOB_QUEUED`. Habilita el modo privilegiado de Docker en el entorno de build porque el job construye imágenes.
4. Asocia un service role de CodeBuild con permisos limitados para usar la conexión, publicar en el repositorio ECR, registrar la definición de tarea, actualizar/describir el servicio ECS y pasar solo el ECS task execution role. CodeBuild usa ese rol para entregar credenciales temporales al runner.
5. Crea RDS PostgreSQL no pública y guarda un secreto JSON en Secrets Manager con `url`, `username` y `password`. `url` debe contener la URL JDBC completa, por ejemplo `jdbc:postgresql://HOST_RDS:5432/ecommerce`. El Security Group de RDS solo debe permitir TCP 5432 desde el Security Group de las tareas.
6. Crea un ECS task execution role con permisos para descargar de ECR, enviar logs a CloudWatch y leer ese secreto. Añade `kms:Decrypt` únicamente si el secreto usa una clave KMS propia. La aplicación actualmente no llama a APIs AWS, así que no necesita permisos adicionales en un task role.
7. Crea el grupo CloudWatch Logs `/ecs/acm-ecommerce` con retención definida.
8. Crea el cluster y servicio ECS/Fargate con una definición de tarea basada en `ecs/task-definition.json`, Linux `X86_64`, `0.5 vCPU / 1 GB` y el puerto `8080`.
9. Usa un ALB **interno** y un target group de tipo `ip`; el health check es `/actuator/health/readiness`. No abras listeners o Security Groups a Internet. El Security Group de tareas debe aceptar tráfico solo desde el Security Group del ALB.

La VPC predeterminada inventariada tiene subredes públicas en tres AZ. Para usar tareas sin IP pública, configura subredes privadas y salida a ECR, Secrets Manager y CloudWatch Logs mediante NAT o VPC endpoints. Ambos enfoques pueden generar cargos; compara los costos antes de elegir. Mantén RDS y las tareas en subredes privadas.

## Completar la definición de tarea

Antes de activar el workflow, edita `ecs/task-definition.json`:

- Sustituye `REPLACE_WITH_ECS_TASK_EXECUTION_ROLE_ARN` por el ARN real del execution role.
- Sustituye cada `REPLACE_WITH_DB_SECRET_ARN` por el ARN del secreto, conservando los sufijos `:url::`, `:username::` y `:password::`.
- Conserva el nombre de contenedor `acm-ecommerce`, el puerto `8080` y la configuración de logs en `us-east-2`.

No incluyas contraseñas ni valores del secreto en GitHub o en el repositorio. ECS los obtiene de Secrets Manager al iniciar cada tarea.

## GitHub Actions

Configura estas variables en **Settings > Secrets and variables > Actions > Variables**:

- `CODEBUILD_RUNNER_PROJECT`: `acm-ecommerce-deploy-runner`.
- `ECR_REPOSITORY`: `acm-ecommerce`.
- `ECS_CLUSTER`: nombre del cluster.
- `ECS_SERVICE`: nombre del servicio Fargate.
- `ECS_DEPLOY_ENABLED`: déjalo sin configurar o en `false` hasta que la conexión GitHub esté `AVAILABLE`, CodeBuild, RDS, secreto, ECS, ALB, roles, permisos y definición de tarea estén listos. Solo entonces establece `true`.

El job de CI usa un PostgreSQL temporal. Para él configura los GitHub Actions secrets `DB_NAME`, `DB_USER` y `DB_PASSWORD`; no pongas en ellos las credenciales de producción. El job de despliegue corre en el CodeBuild runner, publica la imagen en ECR con el SHA del commit y actualiza el servicio ECS. No necesita `id-token: write` ni access keys AWS en GitHub.

GitHub solo ejecuta workflows dentro de `.github/workflows/`. El archivo `build-java.yml` de la raíz es una copia antigua, no el pipeline activo.

## Por qué CodeBuild Runner

El job `deploy-ecs` usa `runs-on: codebuild-${{ vars.CODEBUILD_RUNNER_PROJECT }}-${{ github.run_id }}-${{ github.run_attempt }}`. CodeBuild aporta las credenciales temporales de su service role; el workflow no crea un proveedor IAM OIDC.

Esta ruta evita `iam:CreateOpenIDConnectProvider`, pero **no evita IAM**: el SCP debe permitir crear/usar el service role de CodeBuild, `iam:PassRole` y las acciones de CodeBuild/CodeConnections. Si también deniega esas acciones, habrá que pedir a AWS que habilite el flujo aprobado. CodeBuild factura tiempo de build; revisa el costo antes de usar runners.

Otra posibilidad es mover toda la orquestación a CodePipeline + CodeConnections + CodeBuild. También usa GitHub App y roles de servicio, pero requiere reemplazar el pipeline de GitHub Actions. Ninguna opción usa access keys AWS permanentes en GitHub.

## Acceso y validación

Al ser interno, el DNS del ALB no se podrá abrir desde Internet ni desde una laptop sin conectividad privada a la VPC. Valida el despliegue revisando la salud del target group y los logs de CloudWatch; para hacer peticiones HTTP desde fuera de la VPC necesitarás una conexión privada aprobada, como VPN o un bastion controlado.

La ruta `/actuator/health/readiness` incluye el estado de PostgreSQL. No publiques la API hasta implementar autenticación y autorización: `DisableCsrf.java` permite actualmente todas las rutas sin autenticación y desactiva CSRF.

## Costos

RDS, Fargate, ALB, NAT Gateway o VPC endpoints, logs, almacenamiento ECR y tráfico pueden consumir créditos o generar cargos. Consulta **AWS Settings > Billing** y conserva los recursos solo mientras los necesites.
