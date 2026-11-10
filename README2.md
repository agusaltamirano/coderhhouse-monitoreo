# EduCloud - Plan de Monitoreo y Auditoría

Pre-entrega 7 - AWS Cloud Practitioner (Coderhouse)
Agustín Altamirano

Este plan toma la arquitectura de las entregas anteriores (VPC en 2 AZ, ALB + EC2 en Auto Scaling, RDS PostgreSQL Multi-AZ, S3 + CloudFront) y define qué voy a monitorear, cuándo me tiene que avisar AWS y cómo dejo registrados los cambios en la infraestructura.

## 1. Recursos a monitorear

| Servicio | Por qué lo monitoreo |
|---|---|
| EC2 (Auto Scaling Group) | Es donde corre el backend. Si se satura o se cae, la plataforma deja de responder. |
| Application Load Balancer | Es la puerta de entrada al backend. Muestra errores y tiempos de respuesta que ve el usuario. |
| RDS PostgreSQL | Guarda usuarios, inscripciones y pagos. Quedarse sin espacio o conexiones frena todo. |
| NAT Gateway | Es la salida a internet de las subredes privadas, y cobra por GB procesado. |
| S3 + CloudFront | Sirven los videos y PDFs de los cursos. |
| Facturación | El proyecto tiene presupuesto chico, así que el gasto también es una métrica crítica. |

## 2. Métricas seleccionadas

| Servicio | Métrica | Qué me dice |
|---|---|---|
| EC2 | `CPUUtilization` | Carga del procesador de cada instancia |
| EC2 | `StatusCheckFailed` | Si la instancia o el hardware tienen fallas |
| EC2 | `NetworkIn` / `NetworkOut` | Tráfico que entra y sale de cada instancia |
| EC2 | `mem_used_percent` | Uso de RAM. **Requiere instalar el CloudWatch Agent**, no viene por defecto |
| ALB | `HealthyHostCount` / `UnHealthyHostCount` | Cuántas instancias pasan el health check |
| ALB | `HTTPCode_Target_5XX_Count` | Errores del backend |
| ALB | `TargetResponseTime` | Cuánto tarda en responder la API |
| RDS | `FreeStorageSpace` | Espacio libre en disco (en bytes) |
| RDS | `CPUUtilization` | Carga de la base |
| RDS | `DatabaseConnections` | Conexiones abiertas |
| NAT Gateway | `BytesOutToDestination` | Datos que salen a internet (impacta en el costo) |
| NAT Gateway | `ErrorPortAllocation` | Si el NAT se quedó sin puertos para nuevas conexiones |
| CloudFront | `5xxErrorRate` | Porcentaje de errores al servir contenido |
| S3 | `BucketSizeBytes` | Tamaño del bucket de material (métrica diaria) |
| Billing | `EstimatedCharges` | Gasto estimado del mes |
| Logs | `ApiErrorCount` | Métrica propia, sale de un filtro sobre los logs de la API (ver punto 4) |

## 3. Matriz de alarmas

Todas las alarmas notifican por **Amazon SNS** a dos tópicos:

- `educloud-alertas-criticas`: el servicio está caído o a punto de caer. Envía email a admin@example.com y SMS.
- `educloud-alertas-aviso`: algo para revisar en el día. Solo envía email a admin@example.com.

Para evitar alarmas "ruidosas" uso períodos de 5 minutos y pido que se cumpla la condición en más de un período seguido. Así un pico de un minuto no dispara nada.

| # | Métrica | Condición (umbral) | Período | Acción de alerta |
|---|---|---|---|---|
| 1 | EC2 `CPUUtilization` (promedio del ASG) | > 80% | 5 min, 3 de 3 períodos (15 min) | SNS aviso + política de Auto Scaling que agrega 1 instancia |
| 2 | EC2 `StatusCheckFailed` | >= 1 | 1 min, 2 de 2 períodos | SNS críticas |
| 3 | ALB `UnHealthyHostCount` | >= 1 | 5 min, 2 de 2 períodos | SNS críticas |
| 4 | ALB `HTTPCode_Target_5XX_Count` | > 10 errores | 5 min, 2 de 2 períodos | SNS críticas |
| 5 | ALB `TargetResponseTime` (promedio) | > 1 segundo | 5 min, 3 de 3 períodos | SNS aviso |
| 6 | RDS `FreeStorageSpace` | < 3 GB (15% de los 20 GB) | 5 min, 2 de 2 períodos | SNS críticas |
| 7 | RDS `CPUUtilization` | > 80% | 5 min, 3 de 3 períodos | SNS aviso |
| 8 | RDS `DatabaseConnections` | > 80 conexiones | 5 min, 2 de 2 períodos | SNS aviso |
| 9 | NAT `ErrorPortAllocation` | > 0 | 5 min, 1 período | SNS aviso |
| 10 | NAT `BytesOutToDestination` | > 5 GB en una hora | 1 hora, 1 período | SNS aviso (posible costo inesperado) |
| 11 | CloudFront `5xxErrorRate` | > 5% | 5 min, 2 de 2 períodos | SNS críticas |
| 12 | `ApiErrorCount` (filtro de logs) | > 20 errores | 5 min, 1 período | SNS aviso |
| 13 | `EstimatedCharges` | > 10 USD | 6 horas | SNS aviso |
| 14 | `EstimatedCharges` | > 25 USD | 6 horas | SNS críticas |
| 15 | `SecurityGroupChanges` (filtro de CloudTrail) | >= 1 | 5 min, 1 período | SNS críticas (ver punto 5) |

Notas:

- `FreeStorageSpace` se mide en bytes, no en porcentaje. Por eso el 15% lo pasé a 3 GB, calculado sobre la instancia de 20 GB que definí en la pre-entrega 5.
- La métrica de facturación solo existe en **us-east-1**, y hay que activarla antes en *Billing > Billing preferences > Receive CloudWatch billing alerts*.
- `BucketSizeBytes` de S3 se actualiza una vez por día, así que solo la muestro en el dashboard. No le pongo alarma de 5 minutos.

## 4. Dashboard

Nombre: **EduCloud-Produccion**. Rango por defecto: últimas 24 h.

![Mock-up del dashboard](dashboard.png)

El panel se lee de arriba hacia abajo, de lo más urgente a lo más detallado:

**Fila 1, el resumen.** Es lo primero que miro cuando entro.
- Widget de estado de alarmas: todas las alarmas de la matriz, en verde o rojo.
- Número: hosts sanos detrás del ALB (`HealthyHostCount`, ej. "2 / 2").
- Número: espacio libre en RDS (`FreeStorageSpace`).
- Número: gasto estimado del mes (`EstimatedCharges`).

**Fila 2, cómputo.**
- Gráfico de líneas: `CPUUtilization` de cada EC2 en las últimas 24 h, con una línea horizontal en 80% para ver qué tan cerca está de la alarma.
- Gráfico de líneas: `RequestCount` y `HTTPCode_Target_5XX_Count` del ALB.
- Gráfico de líneas: `TargetResponseTime`.

**Fila 3, red.**
- `NetworkIn` / `NetworkOut` de las EC2.
- `BytesOutToDestination` del NAT Gateway.
- Requests y `5xxErrorRate` de CloudFront.

**Fila 4, base de datos y almacenamiento.**
- `CPUUtilization` y `DatabaseConnections` de RDS en el mismo gráfico.
- `FreeStorageSpace` de los últimos 7 días, para ver la tendencia.
- `BucketSizeBytes` del bucket de S3 en los últimos 30 días.

**Fila 5, logs.**
- Tabla de CloudWatch Logs Insights con los últimos errores de la API. El backend escribe sus logs en el log group `/educloud/api` mediante el CloudWatch Agent.

Consulta del widget de logs:

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

Sobre el mismo log group creo un **metric filter** con el patrón `ERROR`, que genera la métrica `ApiErrorCount` usada en la alarma 12.

## 5. Auditoría con CloudTrail y AWS Config

CloudWatch me dice **cómo está funcionando** la plataforma. CloudTrail me dice **quién hizo qué** en la cuenta. Config me dice si los recursos **cumplen las reglas** que definí.

### CloudTrail

Configuración:
- Un trail llamado `educloud-trail`, **multi-región** y con eventos de administración (lectura y escritura).
- Los logs se guardan en un bucket S3 dedicado (`educloud-cloudtrail-logs`), privado y con cifrado.
- Tiene activada la **validación de integridad** de archivos de log, para detectar si alguien los modificó.
- Con una regla de ciclo de vida en S3, los logs pasan a Glacier a los 90 días.
- El trail también envía los eventos a CloudWatch Logs, para poder crear metric filters y alarmas sobre ellos.

Cómo rastreo quién modificó los Security Groups de la pre-entrega 6 (SG-ALB, SG-APP, SG-DB):

1. En CloudTrail > *Event history* filtro por nombre de evento: `AuthorizeSecurityGroupIngress`, `RevokeSecurityGroupIngress`, `ModifySecurityGroupRules` o `DeleteSecurityGroup`.
2. En cada evento reviso:
   - `userIdentity.arn`: quién lo hizo.
   - `sourceIPAddress`: desde dónde.
   - `eventTime`: cuándo.
   - `requestParameters`: qué regla agregó o sacó. Por ejemplo, si abrió el 5432 a `0.0.0.0/0`.
3. Para no tener que revisar a mano, creo un metric filter sobre el log group del trail:

```
{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) ||
  ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) ||
  ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup) ||
  ($.eventName = ModifySecurityGroupRules) }
```

Ese filtro genera la métrica `SecurityGroupChanges`, que usa la alarma 15. Cualquier cambio en un SG me llega por mail en menos de 5 minutos.

Otros eventos críticos de red que también entran en el filtro de alertas:

| Evento | Por qué es crítico |
|---|---|
| `DeleteVpc` | Borra toda la red de la plataforma |
| `DeleteNatGateway` | Las subredes privadas pierden salida a internet |
| `DetachInternetGateway` / `DeleteInternetGateway` | La aplicación deja de ser accesible |
| `CreateRoute` / `ReplaceRoute` / `DeleteRoute` | Cambia el camino del tráfico; una ruta mal puesta puede exponer una subred privada |
| `CreateNetworkAclEntry` / `ReplaceNetworkAclEntry` | Cambia las reglas de las NACL |
| `ModifyDBInstance` con `PubliclyAccessible=true` | Expone la base a internet |
| `StopLogging` / `DeleteTrail` | Alguien intenta apagar la auditoría |

### AWS Config

Habilito el *configuration recorder* para todos los recursos de la región. El historial se guarda en S3 y las notificaciones de incumplimiento van al tópico SNS de avisos.

Reglas administradas (managed rules) que implemento:

| Regla | Qué controla | Por qué la elijo |
|---|---|---|
| `s3-bucket-public-read-prohibited` (y `s3-bucket-public-write-prohibited`) | Que ningún bucket de S3 sea público | El material de los cursos solo se sirve por CloudFront. Un bucket público expone contenido pago |
| `rds-instance-public-access-check` | Que la instancia RDS no tenga acceso público | La base tiene que quedar solo en las subredes privadas, como en el diagrama de red |
| `restricted-ssh` (incoming-ssh-disabled) | Que ningún Security Group permita el puerto 22 desde `0.0.0.0/0` | Las EC2 están en subred privada, así que no debería haber SSH abierto a internet |

Como extra, también sumaría `cloudtrail-enabled` (verifica que el trail no se apague) y `rds-storage-encrypted`.

Si una regla queda en estado **NON_COMPLIANT**, Config manda la notificación por SNS y el evento queda registrado. Así puedo cruzarlo con CloudTrail y ver quién hizo el cambio.

## Archivos del repositorio

- `README.md`: este plan.
- `dashboard.png`: mock-up del dashboard.
- `dashboard.drawio`: archivo editable del mock-up. Se abre en https://app.diagrams.net
