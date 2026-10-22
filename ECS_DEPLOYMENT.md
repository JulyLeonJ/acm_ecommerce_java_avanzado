# Despliegue privado en ECS/Fargate

La región del proyecto es `us-east-2`. La API se mantendrá privada, como se acordó. La infraestructura está preparada; el servicio ECS permanece en `desiredCount=0` hasta que la versión de este workflow se publique en GitHub y el job de despliegue se habilite.

## Recursos necesarios

1. Existe un ECR privado `acm-ecommerce` con tags inmutables y escaneo básico al subir imágenes.
2. La conexión CodeConnections `acm-ecommerce-github` está `AVAILABLE` para este repositorio.
3. Existe el CodeBuild Runner `acm-ecommerce-deploy-runner`, asociado al evento `WORKFLOW_JOB_QUEUED`, con Docker privilegiado y concurrencia limitada a una ejecución. Su service role publica en ECR, actualiza el servicio ECS y pasa solo el task execution role.
4. Existe RDS PostgreSQL 18.3 `db.t4g.micro`, Single-AZ, 20 GB gp3, cifrada, no pública y con contraseña administrada por RDS/Secrets Manager. El Security Group de RDS acepta TCP 5432 solo desde las tareas.
5. Existe el ECS task execution role con permisos para descargar de ECR, enviar logs y leer el secreto RDS. La aplicación no llama a APIs AWS, así que no necesita permisos adicionales en un task role.
6. Existe `/ecs/acm-ecommerce` en CloudWatch Logs con retención de 7 días.
7. Existen el cluster `acm-ecommerce`, la definición de tarea inicial y el servicio Fargate `acm-ecommerce`. El servicio está en `desiredCount=0` hasta que el pipeline publique una imagen válida.
8. Existe un ALB **interno** y un target group de tipo `ip`; el health check es `/actuator/health/readiness`. No hay listener público. El Security Group de tareas acepta tráfico solo desde el Security Group del ALB.

La VPC predeterminada tenía subredes públicas en tres AZ. Se añadieron subredes privadas en `us-east-2a` y `us-east-2b` con salida por un NAT Gateway en `us-east-2a`. Las tareas no reciben IP pública; RDS y ECS usan las subredes privadas.

## Completar la definición de tarea

`ecs/task-definition.json` contiene el ARN real del execution role, la URL privada de RDS, el usuario y la referencia al secreto administrado por RDS. El workflow solo sustituye la imagen por el SHA del commit.

No incluyas contraseñas ni valores del secreto en GitHub o en el repositorio. ECS los obtiene de Secrets Manager al iniciar cada tarea.

## GitHub Actions

Las siguientes variables ya están configuradas en **Settings > Secrets and variables > Actions > Variables**:

- `CODEBUILD_RUNNER_PROJECT`: `acm-ecommerce-deploy-runner`.
- `ECR_REPOSITORY`: `acm-ecommerce`.
- `ECS_CLUSTER`: `acm-ecommerce`.
- `ECS_SERVICE`: `acm-ecommerce`.
- `ECS_DEPLOY_ENABLED`: actualmente `false`; solo se activará cuando el cambio local del workflow esté en GitHub.

El job de CI usa un PostgreSQL temporal. Ya están configurados los GitHub Actions secrets `DB_NAME`, `DB_USER` y `DB_PASSWORD`; no contienen las credenciales productivas. El job de despliegue corre en el CodeBuild runner, publica la imagen en ECR con el SHA del commit y actualiza el servicio ECS. No necesita `id-token: write` ni access keys AWS en GitHub.

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
