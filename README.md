# Sistema Distribuido en 3 Capas con VirtualBox y Ubuntu Server

## Seminario de Actualización II – Laboratorio

**Alumno:** Fabricio Augusto
**Tema:** Implementación de arquitectura distribuida en 3 capas utilizando máquinas virtuales Linux.

---

# Introducción

El objetivo de este trabajo práctico fue implementar un sistema distribuido real separando los servicios en tres máquinas virtuales diferentes. Cada servidor cumple una función específica dentro de la arquitectura:

* Base de datos
* Backend/API REST
* Frontend web

La implementación se realizó utilizando VirtualBox y Ubuntu Server, configurando comunicación entre máquinas, acceso remoto mediante SSH y persistencia de datos utilizando MariaDB.

---

# Objetivo del Proyecto

Desarrollar un sistema funcional de registro de alumnos utilizando una arquitectura distribuida de 3 capas.

El sistema debía permitir:

* Registrar alumnos
* Validar DNI duplicados
* Consultar alumnos almacenados
* Mantener los datos persistentes
* Separar responsabilidades entre servidores

---

# Arquitectura del Sistema

| Máquina Virtual | Función               | IP        | Puerto |
| --------------- | --------------------- | --------- | ------ |
| VM1             | Base de Datos MariaDB | 10.0.2.15 | 3306   |
| VM2             | Backend Node.js       | 10.0.2.16 | 3454   |
| VM3             | Frontend Apache       | 10.0.2.17 | 8080   |

---

# Flujo del Sistema

```text
Navegador
   ↓
Frontend Apache (VM3)
   ↓
API REST Node.js (VM2)
   ↓
MariaDB (VM1)
```

---

# Tecnologías Utilizadas

| Tecnología          | Uso                   |
| ------------------- | --------------------- |
| Oracle VirtualBox   | Virtualización        |
| Ubuntu Server 24.04 | Sistema Operativo     |
| MariaDB             | Base de datos         |
| Node.js + Express   | Backend               |
| Apache2             | Frontend              |
| HTML + JavaScript   | Interfaz web          |
| SSH                 | Administración remota |
| Netplan             | Configuración de red  |

---

# Creación de las Máquinas Virtuales

Se creó inicialmente una VM base con Ubuntu Server.

## Recursos asignados

* 2 GB RAM
* 2 CPUs
* 20 GB Disco dinámico

Durante la instalación:

* Se instaló OpenSSH Server
* Se utilizó Ubuntu Server completo
* Se configuró usuario administrador

Luego la VM fue clonada para generar:

* VM1-DB
* VM2-BACKEND
* VM3-FRONTEND

Al clonar las máquinas se habilitó la opción:

```text
Generar nuevas direcciones MAC
```

para evitar conflictos de red.

---

# Configuración de Red

Cada máquina virtual fue configurada con red NAT.

## Direcciones IP utilizadas

| VM  | IP        |
| --- | --------- |
| VM1 | 10.0.2.15 |
| VM2 | 10.0.2.16 |
| VM3 | 10.0.2.17 |

---

# Configuración de Netplan

Archivo editado:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Ejemplo de configuración:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.0.2.15/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses:
          - 8.8.8.8
```

Aplicar cambios:

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

---

# Configuración de SSH y Port Forwarding

## Reglas utilizadas

| Servicio        | Puerto Host | Puerto VM |
| --------------- | ----------- | --------- |
| SSH VM1         | 2221        | 22        |
| SSH VM2         | 2222        | 22        |
| SSH VM3         | 2223        | 22        |
| Backend API     | 3454        | 3454      |
| Frontend Apache | 8080        | 8080      |

---

# Acceso remoto por SSH

## VM1

```bash
ssh usuario@localhost -p 2221
```

## VM2

```bash
ssh usuario@localhost -p 2222
```

## VM3

```bash
ssh usuario@localhost -p 2223
```

---

# VM1 – Configuración de MariaDB

## Instalación

```bash
sudo apt update
sudo apt install mariadb-server -y
```

## Creación de Base de Datos

```sql
CREATE DATABASE escuela;
USE escuela;

CREATE TABLE alumnos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    apellidos VARCHAR(100),
    nombres VARCHAR(100),
    dni VARCHAR(20) UNIQUE
);
```

---

# Usuario Remoto

```sql
CREATE USER 'usuario_consulta'@'%' IDENTIFIED BY '1234';
GRANT ALL PRIVILEGES ON escuela.* TO 'usuario_consulta'@'%';
FLUSH PRIVILEGES;
```

---

# Habilitar Conexiones Externas

Editar:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Cambiar:

```text
bind-address = 127.0.0.1
```

por:

```text
bind-address = 0.0.0.0
```

Reiniciar servicio:

```bash
sudo systemctl restart mariadb
```

---

# VM2 – Backend con Node.js

## Instalación

```bash
sudo apt update
sudo apt install nodejs npm -y
```

## Dependencias

```bash
npm init -y
npm install express mysql2 cors
```

---

# Código del Backend

Archivo:

```text
server.js
```

## Funcionalidades implementadas

### POST /grabaAlumnos

* Recibe datos del alumno
* Verifica si el DNI ya existe
* Inserta el alumno si no está repetido
* Devuelve:

  * 1 = éxito
  * 0 = error

### GET /consultarAlumnos

* Devuelve todos los alumnos
* Ordenados por apellido y nombre

---

# Conexión a Base de Datos

```javascript
const db = mysql.createConnection({
    host: '10.0.2.15',
    user: 'usuario_consulta',
    password: '1234',
    database: 'escuela'
});
```

---

# Ejecución del Backend

```bash
node server.js
```

Verificación:

```bash
curl http://localhost:3454/consultarAlumnos
```

---

# VM3 – Frontend con Apache

## Instalación

```bash
sudo apt update
sudo apt install apache2 -y
```

---

# Cambio de Puerto

Editar:

```bash
sudo nano /etc/apache2/ports.conf
```

Cambiar:

```text
Listen 80
```

por:

```text
Listen 8080
```

Editar:

```bash
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Cambiar:

```text
<VirtualHost *:80>
```

por:

```text
<VirtualHost *:8080>
```

Reiniciar Apache:

```bash
sudo systemctl restart apache2
```

---

# Estructura del Frontend

```text
/var/www/html/Sistema
│
├── index.html
└── app.js
```

---

# Funcionalidades del Frontend

La interfaz web permite:

* Registrar alumnos
* Consultar alumnos cargados
* Mostrar mensajes de éxito/error
* Consumir la API mediante Fetch API

---

# Comunicación entre Capas

## Frontend → Backend

```javascript
fetch('http://localhost:3454/grabaAlumnos')
```

## Backend → Base de Datos

```text
10.0.2.15:3306
```

---

# Pruebas Realizadas

## Verificación de red

```bash
ping 10.0.2.15
```

## Verificación de API

```bash
curl http://localhost:3454/consultarAlumnos
```

## Verificación del Frontend

```text
http://localhost:8080/Sistema
```

---

# Problemas Encontrados

## Error con Netplan

El archivo YAML tenía errores de indentación y Netplan no aplicaba la configuración.

### Solución

Revisar cuidadosamente espacios y tabulaciones.

---

## Error de conexión a MariaDB

El backend no podía conectarse a la base de datos.

### Solución

Modificar:

```text
bind-address = 0.0.0.0
```

---

## Error por IP duplicada

Las VMs clonadas tenían la misma IP.

### Solución

Modificar manualmente las IPs en Netplan.

---

## Problemas con permisos de Netplan

Aparecía el warning:

```text
Permissions too open
```

### Solución

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
```

---

# Resultado Final

El sistema quedó funcionando correctamente:

* Frontend operativo en Apache
* Backend respondiendo peticiones
* Base de datos persistiendo información
* Comunicación exitosa entre las 3 capas
* Acceso remoto por SSH funcionando

---

# Conclusión

Este trabajo permitió comprender el funcionamiento de una arquitectura distribuida real utilizando virtualización.

Además, se trabajó con:

* Configuración de redes
* Administración de servidores Linux
* Servicios web
* Bases de datos
* APIs REST
* Comunicación entre capas
* Acceso remoto mediante SSH

La práctica ayudó a entender cómo se conectan y comunican distintos servicios dentro de una infraestructura distribuida.
