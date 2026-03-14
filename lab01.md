En este laboratorio mezclaremos tres conceptos importantes: VNet, Subnets y NSG rules.

**Objetivo:**

Crear:
``` 
VNet-A
10.0.0.0/16

Subnet-App
10.0.1.0/24

Subnet-DB
10.0.2.0/24
```

Una vez creadas la vnet y subnets deberemos de:

1. Crear NSG
2. Permitir tráfico App → DB puerto 1433
3. Bloquear tráfico Internet → DB

Una vez creado y comprendido lo realizado, aprenderemos: NSG rules, Prioridad y tráfico inbound/outbound.

---

# Conceptos empleados:

## Virtual Network:

Azure Virtual Network (VNet) es el servicio de red fundamental de Microsoft Azure que permite a los usuarios crear redes privadas aisladas dentro de la nube. Facilita la comunicación segura entre recursos de Azure, internet y redes locales, proporcionando control total sobre la topología y el tráfico.

## Subnets:

Las subredes de Azure proporcionan una manera de implementar divisiones lógicas dentro de la red virtual. La red se puede segmentar en subredes para ayudar a mejorar la seguridad, aumentar el rendimiento y facilitar la administración.

## Arquitectura 3-tiers simplificada:

La **arquitectura de tres niveles** en el contexto de la nube es una estructura de software que organiza las aplicaciones en tres capas lógicas y físicas: la capa de presentación, la capa de aplicación y la capa de datos.

```
Internet
   │
   ▼
Subnet-App (10.0.1.0/24)
   │
   ▼
Subnet-DB (10.0.2.0/24)
```

## NSG (Network Security Group)

Los grupos de seguridad de red son una manera de limitar el tráfico de red a los recursos de la red virtual. Los grupos de seguridad de red contienen reglas de seguridad que permiten o deniegan el tráfico de red entrante o saliente.

---


# Casos prácticos:

## Caso 1: Permitir tráfico App → DB puerto 1433

Debemos comprender que en este caso el tráfico llega a la base de datos, por tanto, debemos crear una regla **Indbound rule del destino** en nuestro NSG.

- Se crea en el puerto 1433 = SQL Server.

- Configuración de la regla:  

|Campo|Valor|
|--|--|
|Source|IP Addresses|
|Source IP|10.0.1.0/24|
|Destination Port|*|
|Protocol|TCP|
|Action|Allow|
|Priority|100|

Esta regla permite que las aplicaciones puedan conectarse a SQL Server:

```
Origen: 10.0.1.0/24 (Subnet-App)
Destino: Subnet-DB
Puerto: 1433
Acción: Allow
```

## Caso 2: Bloquear Internet → DB

Mismo caso que el anterior, es decir, como el tráfico llega a la base de datos (DB), usaremos **Inbound rule**.

- Configuración de la regla:  

|Campo|Valor|
|--|--|
|Source|Service Tag|
|Source Service Tag|Internet|
|Destination Port|Any|
|Destination Port|*|
|Protocol|Any|
|Action|Deny|
|Priority|200|

## Asociar NSG con Subnet-DB:

Simplemente desde el mismo portal de Azure, accedemos a Subnets y asociamos con la subnet. Así cualquier VM en esa subnet hereda las reglas.

---

# Aclaración:

Azure ya tiene estas reglas por defecto en los **NSG**

|Regla|Acción|
|--|--|
|AllowVNetInBound|Allow|
|AllowAzureLoadBalancerInBound|Allow|
|DenyAllInBound|Deny|

Por tanto, Internet ya está bloqueado y es redundante, pero es un ejemplo de creación para comprender este proceso en un caso lo más simple posible.

--- 

# Esquema del flujo final de tráfico:

```
Internet
   │
   │ (DENY)
   ▼
Subnet-DB

Subnet-App
   │
   │ TCP 1433 (ALLOW)
   ▼
Subnet-DB
```

---

# Propuesta de mejora arquitectónica:

A continuación, vamos a realizar una evolución de la arquitectura 3-tiers creada hacia algo más real.

```
Internet
   │
Azure Load Balancer / App Gateway
   │
Subnet-Web   (10.0.3.0/24) - Web tier
   │
Subnet-App   (10.0.1.0/24) - App tier
   │
Subnet-DB    (10.0.2.0/24) - Database tier
```

---

# Comprobaciones:

Antes de realizar las comprobraciones deben crearse las Virtual Machines que comprobarán las reglas.
- vm-app
- vm-db
- vm-web

## Método 1: Azure Network Watcher.

Mediante el uso de la herramienta **IP Flow Verify** podemos comprobar si una regla permite o bloquea el tráfico.

```
Network Watcher → IP flow verify
```

Configuracíon para verificar, por ejemplo, la regla Allow-App-To-DB-SQL creada:

| Campo       |  Valor  |
| ----------- | ------- |
| Direction   |Outbound |
| Protocol    |  TCP    |
| Local IP    |IP vm-app|
| Local Port  |  *      |
| Remote IP   |10.0.2.4 |
| Remote Port |1433     |

Resultado:

```
Access: Allowed
Security Rule: Allow-App-To-DB-SQL
```

También podemos evaluar directamente la regla del NSG de DB con Inbound de la VM-DB, la configuración sería la inversa.