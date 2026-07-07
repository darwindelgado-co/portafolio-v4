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

**EFS (Elastic File System)** es un sistema de archivos compartido administrado por AWS sobre NFSv4.1 — cuando múltiples EC2 necesitan acceder a los mismos archivos simultáneamente, sin replicar datos.

Aquí te muestro cómo pensarlo, y por qué importa.

<!--more-->

-----

## Por qué debes conocer esto

En el examen SAA-C03, EFS aparece en escenarios donde múltiples servidores comparten contenido (CMS, home directories, datos compartidos). La pregunta real es: **¿cuándo usar EFS, EBS o S3?** Reconocer eso correctamente te lleva a la respuesta.

Ejemplo real:
- *"Tu aplicación necesita un almacenamiento compartido donde 5 servidores escriben y leen los mismos archivos en tiempo real."*
  - Esto grita **EFS** — múltiples EC2, acceso simultáneo, datos que cambian.

- *"Una EC2 necesita un disco rápido solo para ella."*
  - Esto grita **EBS** — un disco adjunto a una sola instancia.

Cuando entiendas dónde encaja cada uno, descubrirás que **EFS tiene un patrón muy claro**: si es Multi-AZ + acceso compartido + datos vivos, es EFS.

-----

## EBS vs EFS vs S3

Necesitas entender dónde encaja EFS en el universo de almacenamiento AWS:

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

Aquí está el poder: Lifecycle Management mueve archivos entre clases según hace cuánto no se acceden, sin que tú hagas nada. Son 3 reglas independientes, cada una opcional:

| Regla | Qué hace | Se dispara por |
|---|---|---|
| **Transition into IA** | Mueve archivos de Standard → IA | Días sin acceso en Standard (7/14/30/60/90) |
| **Transition into Archive** | Mueve archivos a Archive — el tier más barato | Días sin acceso en Standard |
| **Transition into Standard** | Devuelve el archivo a Standard | El **primer acceso** en IA o Archive — no es por tiempo |

**Por qué "Transition into Standard" importa:** sin esta regla, un archivo que bajó a IA o Archive se queda ahí aunque empieces a usarlo seguido otra vez. Con esta regla, el primer acceso lo devuelve a Standard automáticamente.

**Archive** es el tier más barato — pensado para datos de retención/compliance que casi nunca se leen, con mayor latencia que IA.

-----

## Modos de rendimiento

EFS te permite elegir cómo se comporta bajo carga:

| Modo | Cuándo usarlo |
|------|---------------|
| **General Purpose** (default) | La mayoría de casos — web servers, CMS, home dirs |
| **Max I/O** | Miles de EC2 accediendo simultáneamente — Big Data, media processing |

General Purpose es la opción segura — si no sabes cuál elegir, usa General Purpose.

-----

## Modos de throughput

El throughput define cuántos datos por segundo puede servir EFS:

| Modo | Comportamiento |
|------|---------------|
| **Bursting** (default) | Throughput escala con el tamaño del filesystem |
| **Provisioned** | Defines el throughput independiente del tamaño |
| **Elastic** | Escala automáticamente según la carga — recomendado |

**Elastic es lo más práctico:** AWS ajusta automáticamente sin que hagas nada.

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
   - Lifecycle: (ej. move to IA después de 30 días)
   - Throughput: `Elastic`
   - Encriptación at-rest: activada
3. **Network**: revisar que cree un Mount Target por AZ con SG que permita `NFS/2049`
4. **File system policy**:
   - Marcar **"Enforce in-transit encryption for all clients"** — fuerza TLS en tráfico NFS, requiere montar con el **EFS mount helper**
   - Dejar sin marcar "Prevent root access by default" y "Enforce read-only access by default"
   - Dejar sin marcar "Prevent anonymous access"
5. **Create**

-----

### Security Group para NFS

El mount target necesita recibir tráfico NFS (puerto 2049) desde tus EC2:

1. EC2 → **Security Groups** → **Create security group**
2. Name: `training-sg-efs`
3. Inbound: `NFS` | Puerto `2049` | Source: el SG de tus EC2
4. **Create**
5. Volver a EFS → tu filesystem → **Network** → editar mount targets → asignar `training-sg-efs`

{{< figure
  src="./sg-int.png"
  alt="Reglas inbound del security group training-sg-efs permitiendo NFS puerto 2049"
  width="700"
  height="auto"
  class="insert-image"
>}}

-----

### Lanzando las instancias

Dos EC2 en diferentes AZ (AZ-a y AZ-b) con:
- IAM Role: `AmazonSSMManagedInstanceCore`
- Security Group: que permita salida NFS/2049

-----

### Montando el filesystem

En **ambas EC2** vía SSM. Como activamos **"Enforce in-transit encryption"**, el mount manual con `mount -t nfs4` va a fallar — usa el **EFS mount helper**:

```bash
# Instalar amazon-efs-utils
sudo dnf install -y amazon-efs-utils

# Crear punto de montaje
sudo mkdir -p /mnt/efs

# Montar con el helper (usa TLS automáticamente)
sudo mount -t efs fs-XXXXXXXX:/ /mnt/efs
```

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

Observa el `df -h`: EFS aparece con `8.0E` (exabytes) — no es un error, es "elástico". EFS crece según lo que uses. Pagas solo por lo que almacenas.

**En EC2-2**, desde otra AZ:
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

Ambas ven el mismo archivo en tiempo real desde AZ diferentes — eso es EFS. Multi-AZ automático, sin replicación manual.

-----

### Montaje persistente con fstab

Para que monte automáticamente al reiniciar:

```bash
echo "fs-XXXXXXXX:/ /mnt/efs efs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

-----

## Conceptos clave para el examen

| Pregunta | Respuesta |
|---|---|
| ¿Cuántas EC2 pueden montar EFS simultáneamente? | Miles — no hay límite definido |
| ¿EFS funciona en Windows? | No. Para Windows usar FSx for Windows File Server |
| ¿Diferencia entre EFS y EBS? | EBS = disco de una EC2. EFS = filesystem compartido entre muchas EC2 |
| ¿EFS es Multi-AZ? | Sí — mount targets en cada AZ, datos se replican automáticamente |
| ¿Cómo se mueven datos a IA? | Con EFS Lifecycle Management — defines días sin acceso y AWS los mueve |
| ¿Qué puerto usa NFS? | 2049. El SG del mount target debe permitir TCP/2049 desde EC2 |
| ¿Pagas por tamaño o por uso? | Por lo que usas — EFS es elástico, pagas por GB/mes |
| ¿Qué hace "Enforce in-transit encryption"? | Obliga a usar TLS en tráfico NFS, requiere mount helper |
| ¿Cuándo elegir EFS sobre EBS? | Cuando múltiples EC2 necesitan acceso simultáneo a los mismos archivos |
| ¿Se puede acceder a EFS desde fuera de AWS? | Sí — con Direct Connect o VPN |

-----

## Lo que debes recordar

Si tu escenario dice "múltiples servidores necesitan compartir datos en tiempo real", es EFS. Si dice "una EC2 necesita almacenamiento rápido", es EBS. Si dice "almacenar millones de objetos sin estructura", es S3.

En el examen SAA, cuando veas Multi-AZ + acceso compartido + datos que cambian, piensa EFS.
