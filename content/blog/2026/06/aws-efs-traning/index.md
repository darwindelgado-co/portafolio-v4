---
date: '2026-06-12T09:00:00-05:00'
tags: ['aws', 'cloud', 'efs']
title: 'SAA-C03 — EFS: Elastic File System'
slug: 'Elastic-File-System'
---

**EFS (Elastic File System)** es un sistema de archivos compartido administrado por AWS — cuando múltiples EC2 necesitan acceder a los mismos archivos simultáneamente, sin replicar datos.

Aquí te muestro cómo pensarlo, y por qué importa.

<!--more-->

-----

## Por qué debes conocer esto

En el examen SAA-C03, EFS aparece en escenarios donde múltiples servidores comparten contenido. La pregunta real es: **¿cuándo necesitas un filesystem compartido?**

Ejemplo real:
- *"Tu aplicación tiene 5 servidores web que escriben y leen los mismos archivos en tiempo real."* → **EFS**
- *"Necesitas home directories compartidos para usuarios en una granja de servidores."* → **EFS**
- *"Un CMS centralizado donde varios app servers escriben content y attachments."* → **EFS**

Cuando entiendas el patrón: si múltiples máquinas necesitan ver los mismos datos **en tiempo real**, es EFS.

-----

## Qué es EFS realmente

EFS es un **Network File System (NFS)** administrado. Tú no provisiones capacidad, no mantienes servidores. AWS lo gestiona. Lo montas en múltiples EC2 simultáneamente desde diferentes Availability Zones.

Pagas solo por lo que usas — no por capacidad reservada. Si tienes 100GB almacenados, pagas por 100GB. Si baja a 50GB, pagas por 50GB.

Es solo Linux. Para Windows usa FSx for Windows File Server.

-----

## Clases de almacenamiento

EFS tiene 4 clases de almacenamiento optimizadas por costo y frecuencia de acceso:

| Clase | Para qué | Costo | Casos de uso |
|-------|----------|-------|--------------|
| **Standard** | Acceso frecuente | Mayor | Datos vivos, en uso constante |
| **Standard-IA** | Acceso infrecuente (días/semanas) | ~92% más barato | Archivos poco usados, datos históricos |
| **Archive** | Acceso casi nunca (retención/compliance) | El más barato | Backups antiguos, cumplimiento normativo |
| **One Zone** | Una sola AZ, acceso frecuente | Más barato que Standard | Dev/test (no production) |

**Regla:** Standard para empezar. Standard-IA si hay archivos olvidados. Archive si casi nunca los toca nadie. One Zone solo si tienes backup en otra parte.

-----

## EFS Lifecycle Management

Aquí está el poder: **EFS mueve archivos automáticamente** entre clases según cuánto tiempo hace que no se acceden. Tú defines las reglas, AWS las ejecuta.

| Transición | Se dispara por | Resultado |
|-----------|----------------|-----------|
| Move to IA | X días sin acceso | Archivo baja a Standard-IA (92% más barato) |
| Move to Archive | X días sin acceso | Archivo baja a Archive (máximo ahorro) |
| Move to Standard | Primer acceso | Archivo vuelve a Standard al instante |

**Ejemplo:** configuras "move to IA después de 30 días sin acceso". Un archivo que nadie toca en 30 días baja automáticamente a IA. El primer acceso lo devuelve a Standard al instante. Sin intervención manual.

**Por qué "Move to Standard" importa:** sin esta regla, un archivo que bajó a Archive se queda ahí aunque empieces a usarlo seguido otra vez — pagando latencia indefinidamente. Con esta regla, el primer acceso lo restaura automáticamente.

-----

## Modos de rendimiento

| Modo | Para qué | Casos de uso |
|------|----------|--------------|
| **General Purpose** (default) | La mayoría de casos | Web servers, CMS, home directories |
| **Max I/O** | Máximo throughput | Miles de EC2, Big Data, media processing |

**Recomendación:** General Purpose siempre. Solo cambias a Max I/O si mides y compruebas que lo necesitas.

-----

## Modos de throughput

| Modo | Comportamiento | Recomendación |
|------|----------------|---------------|
| **Bursting** (default) | Throughput escala con el tamaño | Archivos pequeños o tráfico variable |
| **Provisioned** | Throughput fijo garantizado | Tráfico predecible y alto |
| **Elastic** | Escala automáticamente según carga | Recomendado para producción |

**Conclusión:** Usa **Elastic**. AWS ajusta automáticamente sin que hagas nada.

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

## Lo que debes tener en cuenta

- **Múltiples EC2 necesitan datos compartidos en tiempo real** → EFS es la respuesta
- **EFS es Multi-AZ automático** — datos se replican sin que hagas nada
- **Lifecycle Management mueve archivos automáticamente** — defines días, AWS ejecuta
- **Elastic throughput** — escala solo lo que necesitas, pagas por uso
- **One Zone es barato pero riesgoso** — si esa AZ cae, pierdes los datos

-----

## Lo que debes recordar

EFS es tu filesystem compartido para infraestructura en AWS. Múltiples máquinas, datos vivos, administrado completamente. 

No para base de datos — para eso RDS o DynamoDB.

EFS es simple: compartir archivos entre EC2, Multi-AZ automático, sin replicación manual.
