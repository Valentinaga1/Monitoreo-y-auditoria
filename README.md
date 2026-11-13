# Plan de monitoreo y auditoría · LearnWithMe

**Pre-entrega 7 · AWS Certified Cloud Practitioner**
Recursos monitoreados: los definidos en las entregas de VPC (Módulo 6) y bases de datos (Módulo 5).

## 1. Recursos y métricas críticas

| Servicio | Métricas de CloudWatch | Por qué es crítica |
|---|---|---|
| EC2 (Auto Scaling) | `CPUUtilization`, `StatusCheckFailed` | Saturación del servidor web y fallos de la instancia |
| Application Load Balancer | `HealthyHostCount`, `HTTPCode_Target_5XX_Count` | Servidores disponibles y errores que ven los estudiantes |
| NAT Gateway | `BytesOutToDestination`, `ErrorPortAllocation` | Tráfico de salida (costo) y fallos de conexión |
| RDS MySQL | `FreeStorageSpace`, `CPUUtilization`, `DatabaseConnections` | Espacio, carga y conexiones de la base de datos |
| S3 | `BucketSizeBytes`, `NumberOfObjects` | Crecimiento del almacenamiento (métrica diaria) |
| Facturación | `EstimatedCharges` | Control del presupuesto mensual |
| CloudWatch Logs | Filtro de métrica `ERROR` en `/learnwithme/app` | Errores de la aplicación y de las APIs |

## 2. Matriz de alarmas

Todas las alarmas notifican al tópico de **Amazon SNS `learnwithme-alertas`**, que tiene suscrito el correo `admin@example.com`. La suscripción se confirma desde el email.

Para evitar alarmas ruidosas, ninguna usa periodos menores a 5 minutos y la mayoría exige varios periodos seguidos.

| Métrica | Condición (umbral) | Periodo | Acción de alerta |
|---|---|---|---|
| EC2 `CPUUtilization` (promedio del grupo) | > 80 % | 3 periodos seguidos de 5 min | Correo a admin@example.com y Auto Scaling añade una instancia |
| EC2 `StatusCheckFailed` | ≥ 1 | 2 periodos de 5 min | Correo a admin@example.com |
| ALB `HealthyHostCount` | < 2 | 2 periodos de 5 min | Correo a admin@example.com |
| ALB `HTTPCode_Target_5XX_Count` | > 10 errores | 2 periodos de 5 min | Correo a admin@example.com |
| NAT `ErrorPortAllocation` | > 0 | 1 periodo de 5 min | Correo a admin@example.com |
| RDS `FreeStorageSpace` | < 3 GiB (15 % de 20 GiB) | 2 periodos de 5 min | Correo a admin@example.com |
| RDS `CPUUtilization` | > 80 % | 3 periodos de 5 min | Correo a admin@example.com |
| Logs: errores de la aplicación | > 20 líneas `ERROR` | 1 periodo de 5 min | Correo a admin@example.com |
| Facturación `EstimatedCharges` | > 10 USD | 6 horas | Correo a admin@example.com |

> **Nota sobre facturación:** la métrica `EstimatedCharges` solo existe en la región **us-east-1**. Antes hay que activar "Recibir alertas de facturación" en la consola de Billing. Además, AWS Budgets permite alertar sobre el gasto **proyectado** del mes, no solo del acumulado.

## 3. Mock-up del dashboard

![Dashboard de CloudWatch de LearnWithMe](Dashboard_CloudWatch_LearnWithMe.png)

El dashboard `LearnWithMe-Produccion` se organiza de arriba hacia abajo, de lo más urgente a lo más detallado:

1. **Estado general:** cuatro widgets numéricos que se ven primero.
   - Instancias sanas en el ALB.
   - Alarmas activas.
   - Gasto estimado del mes.
   - Errores 5XX de la última hora.
2. **Cómputo y red:** dos gráficos de líneas de las últimas 24 h.
   - CPU de cada EC2, con la línea de umbral del 80 %.
   - Tráfico de salida del NAT Gateway.
3. **Base de datos y almacenamiento:** tres gráficos.
   - CPU y conexiones de RDS.
   - Espacio libre de RDS, con la línea de alarma.
   - Tamaño de los buckets S3.
4. **Logs:** un widget de CloudWatch Logs Insights con los últimos errores y advertencias de la aplicación.

## 4. Plan de auditoría

### CloudTrail

- **Trail:** `learnwithme-trail`, activo en **todas las regiones** y con registro de eventos de administración.
- **Almacenamiento:** el bucket privado `learnwithme-logs-prod-use1` (prefijo `cloudtrail/`), con validación de integridad de los archivos activada.
- **Envío a CloudWatch Logs:** cada evento crítico activa una alarma por un filtro de métrica, que avisa al tópico SNS `learnwithme-alertas`.

Cada evento indica **quién** hizo el cambio (`userIdentity`), **cuándo** y **desde qué IP**.

| Evento de CloudTrail | Qué detecta |
|---|---|
| `AuthorizeSecurityGroupIngress` / `ModifySecurityGroupRules` | Alguien abrió o cambió reglas de SG-ALB, SG-Web o SG-DB |
| `DeleteSecurityGroup` | Borrado de un Security Group |
| `CreateNetworkAclEntry` / `ReplaceNetworkAclEntry` | Cambios en las NACL de las subredes |
| `CreateRoute` / `ReplaceRoute` | Cambios en las tablas de ruteo; por ejemplo, una subred privada enviada al IGW |
| `DeleteNatGateway` / `DetachInternetGateway` | Pérdida de conectividad de la VPC |
| `DeleteVpc` | Eliminación de la red completa |
| `ConsoleLogin` con usuario root | Uso de la cuenta root, que no debería ocurrir |

### AWS Config: 3 reglas gestionadas

| Regla | Qué verifica |
|---|---|
| `s3-bucket-level-public-access-prohibited` | Que ningún bucket S3 sea público (Block Public Access activado) |
| `restricted-ssh` | Que ningún Security Group permita SSH (puerto 22) desde 0.0.0.0/0 |
| `rds-instance-public-access-check` | Que la base de datos RDS no sea accesible desde Internet |

AWS Config registra el historial de configuración de cada recurso y marca como **NON_COMPLIANT** los que incumplen una regla.
