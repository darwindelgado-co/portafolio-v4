---
date: '2026-06-12T09:00:00-05:00'
tags: ['aws', 'cloud', 'efs']
title: 'SAA-C03 — EFS: Elastic File System'
slug: 'Elastic-File-System'
---

{{< figure
  src="./Amazon-EFS.png"
  alt="Icono del servicio Amazon EFS"
  width="700"
  height="auto"
  class="insert-image"
>}}

**EFS (Elastic File System)** es un sistema de archivos compartido administrado por AWS que funciona sobre NFSv4.1 — el equivalente a un servidor NFS tradicional, pero sin que tú lo aprovisions ni lo administres. Si necesitas que múltiples instancias EC2 accedan a los mismos archivos simultáneamente, sin replicar datos, EFS es tu solución.

En el examen SAA-C03, EFS aparece constantemente en escenarios donde múltiples servidores comparten contenido (CMS, home directories, datos compartidos). Debes saber cuándo usarlo, cómo se comporta en Multi-AZ, y cómo se diferencia de EBS y S3.

<!--more-->

-----

## EBS vs EFS vs S3

Antes que nada, necesitas entender dónde encaja EFS en el universo de almacenamiento AWS:

| | EBS | EFS | S3 |
|--|-----|-----|----|
| Tipo | Bloque | Archivo (NFS) | Objeto |
| Adjuntar a | 1 EC2 (o 16 con io2) | Múltiples EC2 simultáneamente | No se monta — API/HTTP |
| AZ | 1 AZ | Multi-AZ | Global |
| Sistema de archivos | Tú lo formateas (xfs, ext4) | Ya viene formateado | No aplica |
| Escala | Manual (defines el tamaño) | Automática | Automática |
| OS | Linux y Windows | Solo Linux (para Windows usar FSx) | Cualquiera |
| Caso de uso | Disco de VM | Contenido compartido, CMS, home dirs | Backup, archivos estáticos, data lake |

**Regla de oro:** Si múltiples EC2 necesitan acceso **simultáneo y concurrente** a los mismos archivos, es EFS. Si solo una EC2 necesita almacenamiento rápido, es EBS. Si necesitas almacenar objetos independientes, es S3.

-----

## Clases de almacenamiento EFS

EFS te permite optimizar costos moviendo archivos automáticamente entre niveles según cómo los uses:

| Clase | Para qué | Costo |
|-------|----------|-------|
| **Standard** | Datos de acceso frecuente | Mayor |
| **Standard-IA** | Datos de acceso infrecuente (Infrequent Access) | ~92% más barato |
| **One Zone** | Una sola AZ, acceso frecuente | Más barato que Standard |
| **One Zone-IA** | Una sola AZ, acceso infrecuente | El más barato |

### EFS Lifecycle Management

Aquí está el poder de EFS para **optimizar costos automáticamente**. Lifecycle Management mueve archivos entre clases según hace cuánto no se acceden, sin que tú hagas nada. Son 3 reglas independientes, cada una opcional:

| Regla | Qué hace | Se dispara por |
|---|---|---|
| **Transition into IA** | Mueve archivos de Standard → IA | Días sin acceso en Standard (7/14/30/60/90) |
| **Transition into Archive** | Mueve archivos a Archive — el tier más barato, para datos casi nunca tocados | Días sin acceso en Standard |
| **Transition into Standard** | Devuelve el archivo a Standard | El **primer acceso** (lectura o escritura) en IA o Archive — no es por tiempo |

**Por qué "Transition into Standard" es crítico:** sin esta regla, un archivo que ya bajó a IA o Archive se queda ahí aunque empieces a usarlo seguido otra vez — pagando la latencia de acceso de Archive indefinidamente. Con esta regla, el primer acceso lo devuelve a Standard automáticamente.

**Archive** es el tier más nuevo y más barato (más que IA) — pensado para datos de retención/compliance que casi nunca se leen. A cambio, tiene mayor latencia en el primer byte que IA.

-----

## Modos de rendimiento

EFS te permite elegir cómo se comporta bajo carga:

| Modo | Cuándo usarlo |
|------|---------------|
| **General Purpose** (default) | La mayoría de casos — web servers, CMS, home dirs. Recomendado para empezar |
| **Max I/O** | Miles de EC2 accediendo simultáneamente — Big Data, media processing, cargas de alta contención |

**General Purpose es la opción segura** — si no sabes cuál elegir, usa General Purpose. Max I/O tiene latencia ligeramente mayor pero puede manejar más operaciones concurrentes.

-----

## Modos de throughput

El throughput define cuántos datos por segundo puede servir EFS:

| Modo | Comportamiento | Recomendación |
|------|---------------|---------------|
| **Bursting** (default) | Throughput escala con el tamaño del filesystem — cuanto más datos, más rápido | Archivos pequeños o tráfico variable |
| **Provisioned** | Defines el throughput independiente del tamaño — pagas por lo que garantizas | Tráfico predecible y alto |
| **Elastic** | Escala automáticamente según la carga — maneja picos sin que hagas nada | Recomendado para la mayoría |

**Elastic es lo más práctico:** no tienes que adivinar, AWS ajusta automáticamente.

-----

## Manos a la obra: EFS compartido entre dos EC2

Aquí te muestro cómo construir un EFS compartido y verificar que funciona en tiempo real.

{{< figure
  src="./efs-storage.png"
  alt="Diagrama de red de como se conectan las EC2 y EFS a traves del mount target"
  width="700"
  height="auto"
  class="insert-image"
>}}

### Creando el sistema de archivos

1. **EFS** → **Create file system** → Name: `training-efs` | VPC: `training-vpc`
2. **Customize**: 
   - Storage class: `Standard`
   - Lifecycle: según la tabla de arriba (ej. move to IA después de 30 días)
   - Throughput: `Elastic`
   - Encriptación at-rest: activada (sin costo extra con la key administrada por AWS)
3. **Network**: revisar que cree un Mount Target por AZ, con un Security Group que permita `NFS/2049`
4. **File system policy** (pantalla de "Policy options"):
   - Marcar solo **"Enforce in-transit encryption for all clients"** — fuerza TLS en el tráfico NFS, buena práctica real y tema de examen SAA. Obliga a montar con el **EFS mount helper** (`mount -t efs`).
   - Dejar sin marcar "Prevent root access by default" y "Enforce read-only access by default" — romperían la prueba de escritura que hacemos más adelante en este lab
   - Dejar sin marcar "Prevent anonymous access" — no aplica aquí
5. **Create**

-----

### Security Group para NFS

El mount target necesita recibir tráfico NFS (puerto 2049) desde tus EC2:

1. EC2 → **Security Groups** → **Create security group**
2. Name: `training-sg-efs`
3. Inbound: `NFS` | Puerto `2049` | Source: el SG de tus EC2 (o el CIDR de la VPC)
4. **Create**
5. Volver a EFS → tu filesystem → **Network** → editar los mount targets → asignar `training-sg-efs`

{{< figure
  src="./sg-int.png"
  alt="Reglas inbound del security group training-sg-efs permitiendo NFS puerto 2049"
  width="700"
  height="auto"
  class="insert-image"
>}}

-----

### Lanzando las instancias

Crea dos EC2 en diferentes AZ (recomendado: AZ-a y AZ-b) para simular acceso desde múltiples ubicaciones. Ambas con:
- IAM Role: `AmazonSSMManagedInstanceCore` (para acceder vía SSM Session Manager)
- Security Group: que permita salida NFS/2049 (o usa el mismo SG que asignaste a mount targets)
- OS: Amazon Linux 2

-----

### Montando el filesystem

En **ambas EC2** vía SSM. Como activamos **"Enforce in-transit encryption"** al crear el EFS, el mount manual con `mount -t nfs4` (sin TLS) **va a fallar** — necesitas usar el **EFS mount helper**, que maneja el TLS automáticamente vía `stunnel`:

```bash
# Instalar amazon-efs-utils (contiene el mount helper)
sudo dnf install -y amazon-efs-utils

# Crear punto de montaje
sudo mkdir -p /mnt/efs

# Montar con el helper (usa TLS automáticamente)
sudo mount -t efs fs-XXXXXXXX:/ /mnt/efs

# Verificar que montó correctamente
df -h | grep efs
```

Si montaste correctamente, verás algo como: `fs-XXXXXXXX:/ 8.0E /mnt/efs`

-----

### Verificando que el montaje es compartido

**En EC2-1**, escribe un archivo:
```bash
echo "escrito desde EC2-1" | sudo tee /mnt/efs/prueba.txt
```

{{< figure
  src="./ec2-1-efs-prueba.png"
  alt="Terminal de EC2-1 montando EFS y escribiendo el archivo de prueba"
  width="700"
  height="auto"
  class="insert-image"
>}}

Observa el `df -hT`: el filesystem aparece con `8.0E` (exabytes) de capacidad — no es un error, es justamente lo que significa "elástico". EFS no tiene un tamaño que tú definas; AWS crece según lo que uses. Pagas solo por lo que almacenas, no por capacidad reservada.

**En EC2-2**, lee ese archivo desde una AZ diferente:
```bash
cat /mnt/efs/prueba.txt
# Resultado: escrito desde EC2-1
```

{{< figure
  src="./ec2-2-efs-prueba.png"
  alt="Terminal de EC2-2 leyendo el archivo escrito desde EC2-1, confirmando el filesystem compartido"
  width="700"
  height="auto"
  class="insert-image"
>}}

Ambas ven el mismo archivo en tiempo real desde AZ diferentes — eso es EFS. Sin replicación manual, sin sincronización de datos, sin complejidad. Multi-AZ automático.

-----

### Montaje persistente con fstab

Para que monte automáticamente al reiniciar:

```bash
# Agregar a /etc/fstab
echo "fs-XXXXXXXX:/ /mnt/efs efs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# Verificar que se agregó correctamente
cat /etc/fstab | grep efs
```

De ahora en adelante, cada vez que reinicies la instancia, EFS montará automáticamente.

-----

## Conceptos clave para el examen

| Pregunta | Respuesta |
|---|---|
| ¿Cuántas EC2 pueden montar EFS simultáneamente? | Miles — no hay límite definido oficialmente |
| ¿EFS funciona en Windows? | No. Para Windows usar FSx for Windows File Server |
| ¿Cuál es la diferencia entre EFS y EBS? | EBS = disco de una EC2 (1 a 1). EFS = filesystem compartido entre muchas EC2 (N a N) |
| ¿EFS es Multi-AZ? | Sí — tiene mount targets en cada AZ, los datos se replican automáticamente |
| ¿Cómo se mueven datos a EFS Infrequent Access? | Con EFS Lifecycle Management — defines los días sin acceso y AWS los mueve automáticamente |
| ¿Qué puerto usa NFS? | 2049. El Security Group del mount target debe permitir TCP/2049 desde las EC2 |
| ¿Pagas por el tamaño que defines o por lo que usas? | Por lo que usas — EFS es elástico, no defines capacidad. Pagas por GB/mes |
| ¿Qué hace "Enforce in-transit encryption"? | Obliga a usar TLS en el tráfico NFS, requiere montar con el EFS mount helper |
| ¿Cuándo elegir EFS sobre EBS? | Cuando múltiples EC2 necesitan acceso simultáneo a los mismos archivos |
| ¿Puedo acceder a EFS desde fuera de AWS? | Sí — con AWS Direct Connect o VPN. Por defecto accesible solo desde la VPC |

-----

## Lo que debes recordar

EFS no es un servicio que "actives y olvidas". Es una **decisión arquitectónica**. Si tu escenario dice "múltiples servidores necesitan compartir datos en tiempo real", la respuesta probablemente sea EFS. Si dice "una EC2 necesita almacenamiento rápido", es EBS. Si dice "almacenar millones de objetos sin estructura", es S3.

En el examen SAA, cuando veas Multi-AZ + acceso compartido + datos que cambian, piensa EFS.
