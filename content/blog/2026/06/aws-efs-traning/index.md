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

**EFS (Elastic File System)** es un sistema de archivos compartido administrado por AWS — cuando múltiples EC2 necesitan acceder a los mismos archivos simultáneamente, sin replicar datos.

Aquí te muestro cómo pensarlo, y por qué importa.

<!--more-->

-----

## Por qué debes conocer esto

En el examen SAA-C03, EFS aparece en escenarios donde múltiples servidores comparten contenido. La pregunta real es: **¿cuándo necesitas un filesystem compartido?**

Ejemplo real:
- *"Tu aplicación tiene 5 servidores web que escriben y leen los mismos archivos en tiempo real."* → **EFS** (múltiples EC2, acceso simultáneo, datos vivos)
- *"Necesitas home directories compartidos para usuarios en una granja de servidores."* → **EFS**
- *"Un CMS centralizado donde varios app servers escriben content y attachments."* → **EFS**

Cuando entiendas el patrón: si múltiples máquinas necesitan ver los mismos datos **en tiempo real**, es EFS.

-----

## Qué es EFS realmente

EFS es un **Network File System (NFS)** administrado. Tú no provisiones capacidad, no mantienes servidores. AWS lo gestiona. Lo montas en múltiples EC2 simultáneamente desde diferentes Availability Zones.

Pagas solo por lo que usas — no por capacidad reservada. Si tienes 100GB almacenados, pagas por 100GB. Si baja a 50GB, pagas por 50GB.

Es solo Linux. Para Windows usa FSx for Windows File Server.

-----

## Clases de almacenamiento y Lifecycle

EFS tiene **4 clases de almacenamiento** optimizadas por costo y frecuencia de acceso.

**Standard** — acceso frecuente, costo normal. Aquí empiezas.

**Standard-IA (Infrequent Access)** — datos que nadie toca hace días/semanas, ~92% más barato. Acceso cuesta un poco más, pero el almacenamiento es mucho más barato.

**Archive** — datos que casi nunca se leen (retención, compliance), el más barato. Latencia mayor en el primer byte.

**One Zone** — igual que Standard pero en una sola AZ, más barato. **Riesgo:** si esa AZ cae, pierdes los datos. Solo úsalo si puedes recuperarlos desde backup.

### EFS Lifecycle Management

El poder está aquí: **EFS mueve archivos automáticamente** entre clases según cuánto tiempo hace que no se acceden. Tú defines las reglas, AWS las ejecuta.

**Ejemplo:** configuras "move to IA después de 30 días sin acceso". Un archivo que nadie toca en 30 días baja automáticamente a IA (92% más barato). El primer acceso lo devuelve a Standard al instante. Sin intervención manual.

Tres reglas independientes, todas opcionales:
- Move to IA después de X días sin acceso
- Move to Archive después de X días sin acceso
- Move to Standard en el primer acceso (crítico — sin esto, un archivo en Archive se queda ahí pagando latencia)

-----

## Rendimiento: General Purpose vs Max I/O

**General Purpose** (default) — web servers, CMS, home directories, cualquier caso normal. Si no sabes qué elegir, aquí va.

**Max I/O** — miles de EC2 accediendo simultáneamente, Big Data, media processing. Mayor throughput, mayor latencia. Solo si mides y compruebas que lo necesitas.

Usa General Purpose. Punto.

-----

## Throughput: Bursting, Provisioned, Elastic

El throughput es cuántos datos por segundo puede servir EFS.

**Bursting** (default) — throughput escala con el tamaño del filesystem. Cuanto más datos almacenados, más rápido sirve. Predecible pero limitado.

**Provisioned** — defines un throughput fijo, pagas por garantía. Úsalo si necesitas rendimiento garantizado y consistente.

**Elastic** (recomendado) — escala automáticamente según la demanda. Picos de tráfico, AWS sube. Baja demanda, AWS baja. Pagas lo justo. No tienes que pensar en esto.

Usa Elastic en producción.

-----

## Construyendo tu EFS

Aquí el paso a paso. Vamos a montar un EFS compartido entre dos EC2 en diferentes AZ.

{{< figure
  src="./efs-storage.png"
  alt="Diagrama de red de como se conectan las EC2 y EFS a traves del mount target"
  width="700"
  height="auto"
  class="insert-image"
>}}

### 1. Crea el filesystem

EFS → Create file system → Name: `training-efs`, VPC: `training-vpc`

Customization: Storage class Standard, Throughput Elastic, encriptación at-rest activada.

En **File system policy**, marca **"Enforce in-transit encryption for all clients"** — obliga TLS en tráfico NFS. Datos protegidos en tránsito. Sin esto, los datos van en texto plano por la red.

### 2. Security Group para NFS

El mount target necesita recibir tráfico NFS en puerto 2049 desde tus EC2.

EC2 → Security Groups → Create security group. Name: `training-sg-efs`. 

Inbound rule: NFS, puerto 2049, source: SG de tus EC2.

Vuelve a EFS → tu filesystem → Network → edita los mount targets → asigna `training-sg-efs`.

{{< figure
  src="./sg-int.png"
  alt="Reglas inbound del security group training-sg-efs permitiendo NFS puerto 2049"
  width="700"
  height="auto"
  class="insert-image"
>}}

### 3. Lanza dos EC2 en AZ diferentes

AZ-a y AZ-b. Ambas con IAM Role `AmazonSSMManagedInstanceCore` para acceder vía SSM Session Manager. OS: Amazon Linux 2.

### 4. Monta el filesystem en ambas

Vía SSM en cada EC2:

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/efs
sudo mount -t efs fs-XXXXXXXX:/ /mnt/efs
```

El mount helper maneja TLS automáticamente (porque activamos "Enforce in-transit encryption").

### 5. Verifica que funciona

**En EC2-1**, escribe:
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

Nota el `df -h`: EFS aparece con `8.0E` capacidad — es "elástico", crece con lo que uses. Pagas por GB/mes, no por capacidad reservada.

**En EC2-2** (otra AZ), lee:
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

Mismo archivo desde diferentes AZ, en tiempo real. Eso es EFS. Multi-AZ automático.

### 6. Monta persistente (opcional)

Para que monte al reiniciar:
```bash
echo "fs-XXXXXXXX:/ /mnt/efs efs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

-----

## Lo que debes recordar

**En el examen:** cuando veas "múltiples servidores comparten datos en tiempo real", es EFS. Multi-AZ, elástico, seguro, sin replicación manual.

**En producción:** EFS es tu filesystem compartido. Úsalo para home directories, CMS centralizados, datos compartidos entre aplicaciones. No úsalo para base de datos — para eso está RDS o DynamoDB.

EFS es simple: filesystem compartido, múltiples EC2, datos vivos, administrado por AWS.
