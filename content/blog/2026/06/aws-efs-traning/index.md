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

En el examen SAA-C03, EFS aparece en escenarios donde múltiples servidores comparten contenido. La pregunta real es: **¿cuándo usar EFS, EBS o S3?**

Ejemplo real:
- *"Tu aplicación necesita almacenamiento compartido donde 5 servidores escriben y leen los mismos archivos en tiempo real."* → **EFS** (múltiples EC2, acceso simultáneo, datos vivos)
- *"Una EC2 necesita un disco rápido solo para ella."* → **EBS** (disco adjunto a una sola instancia)
- *"Necesitas almacenar millones de objetos independientes."* → **S3** (API/HTTP, no se monta, global)

Cuando entiendas el patrón, descubrirás: si es Multi-AZ + acceso compartido + datos que cambian, es EFS.

-----

## EBS vs EFS vs S3

**EBS** es un bloque — tú lo formateas con xfs, ext4, lo que quieras. Lo adjuntas a una EC2 y listo. Si necesitas más de una EC2 accediendo, no funciona. Pagas por lo que reservas, aunque no lo uses.

**EFS** es un filesystem formateado por AWS, listo para montar. Múltiples EC2 lo montan simultáneamente desde diferentes AZ. Escala automáticamente — pagas solo por lo que usas. Solo funciona en Linux.

**S3** es almacenamiento de objetos — no se monta como un disco. Lo accedes vía API/HTTP. Global, automático, perfecto para backups y data lakes. Cualquier SO puede acceder.

**Regla de oro:** múltiples EC2 compartiendo archivos = EFS. Una EC2 con disco rápido = EBS. Almacenar objetos masivos = S3.

-----

## Clases de almacenamiento EFS

EFS tiene 4 opciones. Empiezas con **Standard** — datos de acceso frecuente, costo mayor.

Si tienes archivos que nadie toca hace semanas, **EFS Lifecycle Management** los mueve automáticamente a **Standard-IA** (92% más barato). El primer acceso los devuelve a Standard sin que hagas nada.

**Archive** es el tier más barato — para datos de retención/compliance que casi nunca se leen. Latencia mayor, pero costo mínimo.

**One Zone** es más barato que Standard, pero almacena en una sola AZ. Si esa AZ cae, pierdes los datos. Solo úsalo si puedes recuperar esos datos desde otra parte.

Conclusión: empieza con Standard + Elastic throughput, déjale manejar Lifecycle Management. Simple.

-----

## Rendimiento: General Purpose vs Max I/O

**General Purpose** es el default — web servers, CMS, home directories. Si no sabes qué elegir, aquí va.

**Max I/O** es para miles de EC2 accediendo simultáneamente — Big Data, media processing. Mayor throughput, pero más complejo.

Usa General Purpose. Cambias solo si mides y ves que necesitas más.

-----

## Throughput: Bursting, Provisioned, Elastic

**Bursting** escala con el tamaño del filesystem — cuanto más datos, más rápido. Default pero predecible.

**Provisioned** defines el throughput — pagas garantía.

**Elastic** escala automáticamente según la carga. Recomendado. AWS ajusta sin que hagas nada.

-----

## Construyendo tu EFS

Aquí te muestro los pasos, sin complicaciones.

{{< figure
  src="./efs-storage.png"
  alt="Diagrama de red de como se conectan las EC2 y EFS a traves del mount target"
  width="700"
  height="auto"
  class="insert-image"
>}}

### 1. Crea el filesystem

EFS → Create file system → Name: `training-efs`, VPC: `training-vpc`

En customization: Storage class Standard, Throughput Elastic, encriptación activada.

En File system policy, marca **"Enforce in-transit encryption for all clients"** — obliga TLS en tráfico NFS. Esto es importante: sin TLS, los datos van sin protección. Con TLS, necesitas montar con el **EFS mount helper** en lugar del mount manual.

### 2. Security Group para NFS

El mount target necesita recibir tráfico NFS en puerto 2049 desde tus EC2.

EC2 → Security Groups → Create. Name: `training-sg-efs`. Inbound: NFS, puerto 2049, source: el SG de tus EC2.

Vuelve a EFS → tu filesystem → Network → edita los mount targets → asigna `training-sg-efs`.

{{< figure
  src="./sg-int.png"
  alt="Reglas inbound del security group training-sg-efs permitiendo NFS puerto 2049"
  width="700"
  height="auto"
  class="insert-image"
>}}

### 3. Lanza dos EC2 en AZ diferentes

AZ-a y AZ-b. Ambas con IAM Role `AmazonSSMManagedInstanceCore` para acceder vía SSM Session Manager.

### 4. Monta el filesystem

En **ambas EC2** vía SSM. Como activamos "Enforce in-transit encryption", el mount manual falla. Usa el **EFS mount helper**:

```bash
sudo dnf install -y amazon-efs-utils
sudo mkdir -p /mnt/efs
sudo mount -t efs fs-XXXXXXXX:/ /mnt/efs
```

El mount helper maneja TLS automáticamente.

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

Nota el `df -h`: EFS aparece con `8.0E` capacidad — no es error, es "elástico". Crece con lo que uses. Pagas por GB/mes, no por capacidad reservada.

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

Mismo archivo desde diferentes AZ. Eso es EFS. Multi-AZ automático, sin replicación manual, sin sincronización. Tiempo real.

### 6. Monta persistente (opcional)

Para que monte al reiniciar:
```bash
echo "fs-XXXXXXXX:/ /mnt/efs efs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
```

-----

## Lo que debes recordar

**En el examen:** cuando veas "múltiples servidores comparten datos en tiempo real", es EFS. Cuando veas "una EC2 necesita disco rápido", es EBS. Cuando veas "almacenar objetos masivos", es S3.

**En producción:** EFS es Multi-AZ automático, elástico, seguro. Úsalo cuando múltiples servidores necesiten los mismos datos. No uses One Zone a menos que tengas backup en otra parte.

EFS es simple una vez lo entiendes: filesystem compartido, múltiples EC2, datos vivos, sin configuración.
