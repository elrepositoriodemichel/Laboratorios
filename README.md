# Laboratorios

## Propósito

Este repositorio documenta, de forma progresiva, los laboratorios prácticos que mantengo sobre administración de SQL Server en entornos Windows Server con Active Directory Domain Services (AD DS). No es una colección estática de procedimientos: cada laboratorio se despliega, se somete a pruebas de resiliencia (failover, caída de nodos, pérdida de rutas de almacenamiento, etc.) y se documenta junto con los scripts de PowerShell y T-SQL necesarios para replicarlo.

El objetivo es triple: 
- Profundizar en la administración general de SQL Server, tanto on-premises como en Azure, en línea con el temario de DP-300 (Administering Microsoft Azure SQL Solutions).
- Consolidar el conocimiento sobre la infraestructura híbrida de Windows Server que sostiene esos despliegues (WSFC, AD DS, almacenamiento).
- Documentar los escenarios de forma clara y reproducible para que cualquiera pueda contrastarlos o replicarlos.

> **Nota sobre alcance:** los laboratorios y artículos combinan infraestructura on-premises de Windows Server (AD DS, WSFC, almacenamiento, gMSA, delegación Kerberos) con administración de SQL Server y no están centrados únicamente en Azure SQL.



## Host físico (recurso común a todos los laboratorios)

Especificaciones del equipo físico sobre el que corren todos los laboratorios de este repositorio.

|`HARDWARE`||
|---|---|
|**Procesador**| 13th Gen Intel(R) Core(TM) i9-13900K (3.00 GHz) |
|**RAM**| 128 GB (4.200 MT/s) |
|**Almacenamiento** | 4 + 2 + 2 TB SSD (NVMe) + 8 TB HDD (SATA) |
|**Adaptadores de Red** | 2 Adaptadores Wifi (802.11ax) + 1 Adaptador Ethernet|
|`SOFTWARE`||
|**Sistema Operativo** | Windows 11 Pro |
|**Idioma** | Español (España) |

> El i9-13900K es un único socket con die monolítico, por lo que no hay topología NUMA física en este host. Las VM para SQL de mis laboratorios serán configuradas con **nodos NUMA virtuales** en Hyper⁠-⁠V, con propósito de practicar el comportamiento lógico de SQL Server frente a una topología NUMA.

## Base común de los laboratorios

La base de la que parten los laboratorios de WSFC/FCI es:

- Un dominio de Active Directory (**DC**).
- Un servidor de archivos inicialmente para backups (**FS**).
- Un cliente para pruebas de conexión.
- Un **Windows Server Failover Cluster (WSFC) de 2 nodos ya formado**, sin ninguna instancia de SQL Server instalada todavía.

El primer laboratorio está diseñado para simular un entorno con recursos limitados:

- **Windows Server Standard Edition**, que no incluye Storage Spaces Direct (S2D).
- **SQL Server Standard Edition**, que solo permite 2 nodos en una instancia FCI.

Por eso el laboratorio base se construye con dos nodos SQL, replicando un escenario habitual en empresas con licenciamiento Standard.

Posteriormente en el siguiente laboratorio incluiremos un tercer nodo considerando la disponibilidad de las licencias:

- **Windows Server Datacenter Edition** necesaria para S2D.
- **SQL Server Enterprise Edition** que permita FCI de tres nodos.

Tampoco desarrollo en la parte común el **testigo de quórum (witness)** al ser una variable que cambia entre laboratorios (testigo de disco en el Laboratorio 1 con iSCSI, testigo de archivo compartido para compatibilidad con S2D en el Laboratorio 2). Por el mismo motivo, la **topología de almacenamiento** (iSCSI, S2D, etc.) no aparece aquí y será especificada en cada laboratorio según corresponda.

### VMs de la base

| VM | Rol | vCPU | Memoria | Disco C |
|---|:---:|:---:|:---:|:---:|
| SRV⁠-⁠DC01 | AD DS / DNS | 4 vCPU | mín 2GB<br>inicio 4GB<br>máx 8GB | 80GB |
| SRV⁠-⁠FS01 | File Server<br>para Backups | 4 vCPU | mín 2GB<br>inicio 4GB<br>máx 8GB | 80GB |
| SRV⁠-⁠SQL01<br>SRV⁠-⁠SQL02 | Nodos WSFC | 2 NUMA × 4⁠ vCPU <br>8 en total | Estática 16GB | 80GB |
| Win11⁠-⁠CL01 | Cliente | 4 vCPU | mín 2GB<br>inicio 4GB<br>máx 8GB | 128GB |

> **Reserva de CPU:** Estrategia adoptada: **Reserve = 0% por defecto** en los 3 nodos SQL; se elevará al 100% únicamente en el nodo propietario activo durante las ventanas de test que lo requiriesen, utilizando *`Set-VMProcessor -VMName <nodo> -Reserve 100`*, y revirtiéndolo al terminar. Esto libera espacio real para que DC, FS, cliente y el almacenamiento del laboratorio en curso convivan sin contención.

### Red de la base (Management + Heartbeat)

| Servidor | Rol | Management (192.168.5.0/24) | Heartbeat (192.168.10.0/24) |
|---|---|:---:|:---:|
| SRV-DC01 | AD DS / DNS | 192.168.5.11 | - |
| SRV-FS01 | File Server | 192.168.5.13 | - |
| SRV-SQL01 | Nodo WSFC | 192.168.5.21 | 192.168.10.21 |
| SRV-SQL02 | Nodo WSFC | 192.168.5.22 | 192.168.10.22 |
| SRV-SQL03 | Nodo WSFC | 192.168.5.23 | 192.168.10.23 |
| CLU-WSFC | VNN (Cluster) | 192.168.5.50 | - |
| Win11-CL01 | Client | 192.168.5.31 | - |

> Las redes de almacenamiento (Storage01/02-Switch) y sus IPs no forman parte de la base: cada laboratorio las define según su propio backend de almacenamiento.

## Estructura del repositorio

La documentación se organiza en **tres niveles**:

1. **Este documento (general):** contexto, host físico, base común, índice de laboratorios y artículos transversales.
2. **Documentos de laboratorio:** cada laboratorio documenta un escenario completo (topología de red, VMs, roles y evidencias de las pruebas de resiliencia realizadas). Un laboratorio nuevo puede **ampliar** los recursos de uno anterior o **partir de un escenario independiente**; en cualquier caso, el propio documento lo indicará explícitamente al inicio.
3. **Artículos comunes:** desarrollos conceptuales o técnicos que aplican a más de un laboratorio (ej.: creación y uso de las gMSA), para no duplicar la misma explicación en cada laboratorio que los use.

### Índice de laboratorios

| # | Laboratorio | Estado | Contenido |
|:---:|---|:---:|---|
| 1 | [SQL Server FCI con iSCSI](docs/laboratorios/Laboratorio-01.md) | ✅ Publicado | Instancia SQL Server FCI sobre iSCSI con MPIO y testigo de disco. WSFC con dos nodos |
| 2 | Segunda instancia FCI sobre S2D | 🔜 Planificado | Segunda instancia SQL Server FCI, sobre el mismo WSFC sumando el tercer nodo, cambiando a almacenamiento S2D y el testigo a archivo compartido |
| 3 | Alta disponibilidad con Availability Groups | 🔜 Planificado | — |

### Artículos comunes

| Artículo | Estado | Aplica a |
|---|:---:|---|
| gMSA (Group Managed Service Accounts) | 🔜 Pendiente | Cuentas de servicio de clúster/SQL Server en cualquier laboratorio con AD DS |
| MPIO (Multipath I/O) | 🔜 Pendiente | Cualquier laboratorio con más de una ruta física al almacenamiento compartido |

## 🧪 Metodología de validación

No doy por válido un laboratorio por el simple hecho de que los servicios arranquen. Cada laboratorio incluye, como mínimo:

- Prueba de **failover planificado** (`Move-ClusterGroup`) y **no planificado** (apagado forzado de un nodo).
- Verificación de **pérdida de una ruta de almacenamiento** sin interrupción del servicio.
- Comprobación de que los recursos del clúster conmutan correctamente y de que la aplicación cliente reconecta.
- Restauración de base de datos como validación funcional mínima tras cada despliegue y pruebas de backup/restore.

## 🗺️ Hoja de ruta

Este es un laboratorio vivo. Plan inmediato sobre la base de 2 nodos WSFC iniciales ya definida arriba:

1. **Laboratorio 1:** instancia SQL Server FCI sobre almacenamiento iSCSI con MPIO y testigo de disco.
2. **Laboratorio 2:** segunda instancia SQL Server FCI, independiente, sobre el mismo WSFC, cambiando a almacenamiento S2D y testigo de archivo compartido.
3. **Laboratorio 3:** Availability Groups.

Más adelante añadiré certificados TLS y delegación Kerberos. Cada ampliación se reflejará como un nuevo laboratorio o como una revisión explícita y señalizada del existente, **nunca como una reescritura de lo existente**.