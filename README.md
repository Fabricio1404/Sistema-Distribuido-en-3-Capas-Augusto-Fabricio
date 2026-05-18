# Sistema Distribuido de 3 Capas en VMs

**Materia:** Seminario De Actualización II 2026  
**Alumno:** *Fabricio Augusto*  
**Fecha:** Mayo 2026

---

## Descripción del Proyecto

Implementación de un sistema de registro de alumnos distribuido en 3 máquinas virtuales Linux (Ubuntu 22.04) corriendo en VirtualBox, cada una con una responsabilidad específica:

| VMs | Rol | IP NAT | IP Interna | Puerto |
|---|---|---|---|---|
| VM1-BaseDatos | Base de datos MariaDB | 10.0.2.15 | 192.168.56.10 | 3306 |
| VM2-Backend | API REST Node.js/Express | 10.0.2.16 | 192.168.56.11 | 3454 |
| VM3-Frontend | Servidor Web Apache | 10.0.2.17 | 192.168.56.12 | 8080 |

---

## Flujo de Datos

```
Navegador (Host)
      │
      │ localhost:8080
      ▼
VM3 - Apache (10.0.2.17)
      │
      │ 192.168.56.11:3454
      ▼
VM2 - Node.js/Express API (10.0.2.16)
      │
      │ 192.168.56.10:3306
      ▼
VM1 - MariaDB (10.0.2.15)
```

---

## Infraestructura y Red

### Configuración de Red en VirtualBox

Cada VM tiene 2 adaptadores de red:

- **Adaptador 1 (NAT):** Para acceso SSH desde el host y salida a internet
- **Adaptador 2 (Red Interna - intnet):** Para comunicación entre VMs

### Reglas de Port Forwarding (NAT)

**VM1 - Base de Datos:**

| Nombre | Puerto Host | Puerto Invitado |
|---|---|---|
| SSH_DB | 2215 | 22 |
| MySQL_DB | 3306 | 3306 |

**VM2 - Backend:**

| Nombre | Puerto Host | Puerto Invitado |
|---|---|---|
| SSH_BACK | 2216 | 22 |
| API_SERVICE | 3454 | 3454 |

**VM3 - Frontend:**

| Nombre | Puerto Host | Puerto Invitado |
|---|---|---|
| SSH_FRONT | 2217 | 22 |
| WEB_SISTEMA | 8080 | 8080 |

---

## Paso a Paso de Implementación

### FASE 1 — Creación de las VMs

1. Descargar Ubuntu Server 22.04 LTS desde https://releases.ubuntu.com
2. En VirtualBox crear nueva VM con estas especificaciones:
   - RAM: 2048 MB
   - Disco: 10 GB (VDI, reservado dinámicamente)
   - Tipo: Linux 64-bit
3. En Almacenamiento montar el ISO descargado
4. Instalar Ubuntu Server con las siguientes opciones:
   - Idioma: English
   - Teclado: English (US)
   - Tipo: Ubuntu Server (no minimized)
   - Tildar **Install OpenSSH server**
5. Una vez instalada la VM1, clonarla para crear VM2 y VM3:
   - Clic derecho → Clonar
   - Política MAC: Generar nuevas direcciones MAC
   - Tipo: Clonado completo

---

### FASE 2 — Configuración de Red

En cada VM editar el archivo de red:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

**VM1:**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 10.0.2.15/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses:
          - 8.8.8.8
    enp0s8:
      addresses:
        - 192.168.56.10/24
```

**VM2 (cambiar IPs):**
```yaml
    enp0s3:
      addresses:
        - 10.0.2.16/24
    enp0s8:
      addresses:
        - 192.168.56.11/24
```

**VM3 (cambiar IPs):**
```yaml
    enp0s3:
      addresses:
        - 10.0.2.17/24
    enp0s8:
      addresses:
        - 192.168.56.12/24
```

Aplicar cambios en cada VM:

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

Verificar conectividad entre VMs (desde VM2):

```bash
ping -c 3 192.168.56.10
```

Resultado esperado: `0% packet loss`

---

### FASE 3 — VM1: Servidor de Base de Datos

#### Instalación de MariaDB

```bash
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl status mariadb
```

#### Creación de la base de datos y tabla

```bash
sudo mariadb
```

```sql
CREATE DATABASE gestion_alumnos;
USE gestion_alumnos;

CREATE TABLE alumnos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  apellidos VARCHAR(100) NOT NULL,
  nombres VARCHAR(100) NOT NULL,
  dni VARCHAR(20) UNIQUE
);
```

#### Creación del usuario remoto

```sql
CREATE USER 'fabricio'@'192.168.56.11' IDENTIFIED BY 'tu_password';
GRANT ALL PRIVILEGES ON gestion_alumnos.* TO 'fabricio'@'192.168.56.11';
FLUSH PRIVILEGES;
EXIT;
```

#### Habilitar acceso remoto

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Buscar `bind-address = 127.0.0.1` y cambiar por:

```
bind-address = 0.0.0.0
```

```bash
sudo systemctl restart mariadb
```

---

### FASE 4 — VM2: Servidor Backend (Node.js/Express)

#### Instalación de Node.js

```bash
sudo apt update
sudo apt install nodejs npm -y
node -v
npm -v
```

#### Crear el proyecto

```bash
mkdir ~/app-alumnos
cd ~/app-alumnos
npm init -y
```

#### Instalar dependencias

```bash
npm install express cors mysql2
```

#### Verificar instalación

```bash
node -e "require('express'); require('mysql2'); console.log('todo ok')"
```

#### Código de la API (`~/app-alumnos/app.js`)

```js
const express = require('express');
const cors = require('cors');
const mysql = require('mysql2');

const app = express();
app.use(cors());
app.use(express.json());

function getConnection() {
    return mysql.createConnection({
        host: '192.168.56.10',
        user: 'fabricio',
        password: 'tu_password',
        database: 'gestion_alumnos'
    });
}

app.post('/grabaAlumnos', (req, res) => {
    const { apellidos, nombres, dni } = req.body;
    const conn = getConnection();

    conn.query(
        'SELECT COUNT(*) AS count FROM alumnos WHERE dni = ?',
        [dni],
        (err, results) => {
            if (err) {
                conn.end();
                return res.json({ resultado: 0, mensaje: err.message });
            }

            if (results[0].count > 0) {
                conn.end();
                return res.json({ resultado: 0, mensaje: 'DNI ya existe' });
            }

            conn.query(
                'INSERT INTO alumnos (apellidos, nombres, dni) VALUES (?, ?, ?)',
                [apellidos, nombres, dni],
                (err2) => {
                    conn.end();
                    if (err2) {
                        return res.json({ resultado: 0, mensaje: err2.message });
                    }
                    return res.json({ resultado: 1, mensaje: 'Alumno grabado correctamente' });
                }
            );
        }
    );
});

app.get('/consultarAlumnos', (req, res) => {
    const conn = getConnection();

    conn.query(
        'SELECT * FROM alumnos ORDER BY apellidos ASC, nombres ASC',
        (err, results) => {
            conn.end();
            if (err) {
                return res.json({ error: err.message });
            }
            return res.json(results);
        }
    );
});

app.listen(3454, '0.0.0.0', () => {
    console.log('API corriendo en puerto 3454');
});
```

#### Configurar la API como servicio del sistema

```bash
sudo nano /etc/systemd/system/api-alumnos.service
```

```ini
[Unit]
Description=API Alumnos Node.js
After=network.target

[Service]
User=fabricio1404
WorkingDirectory=/home/fabricio1404/app-alumnos
ExecStart=/usr/bin/node /home/fabricio1404/app-alumnos/app.js
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable api-alumnos
sudo systemctl start api-alumnos
sudo systemctl status api-alumnos
```

---

### FASE 5 — VM3: Servidor Frontend (Apache)

#### Instalación de Apache

```bash
sudo apt update
sudo apt install apache2 -y
```

#### Cambiar el puerto a 8080

```bash
sudo nano /etc/apache2/ports.conf
```

Cambiar `Listen 80` por `Listen 8080`

```bash
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Cambiar `<VirtualHost *:80>` por `<VirtualHost *:8080>`

```bash
sudo systemctl restart apache2
```

#### Crear la carpeta y el archivo del sistema

```bash
sudo mkdir /var/www/html/Sistema
sudo nano /var/www/html/Sistema/index.html
```

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Gestión de Alumnos</title>
</head>
<body>
    <h1>Registro de Alumnos</h1>

    <h2>Agregar Alumno</h2>
    <input type="text" id="apellidos" placeholder="Apellidos">
    <input type="text" id="nombres" placeholder="Nombres">
    <input type="text" id="dni" placeholder="DNI">
    <button onclick="grabarAlumno()">Guardar</button>
    <p id="mensaje"></p>

    <h2>Lista de Alumnos</h2>
    <button onclick="consultarAlumnos()">Actualizar Lista</button>
    <table border="1">
        <thead>
            <tr>
                <th>ID</th><th>Apellidos</th><th>Nombres</th><th>DNI</th>
            </tr>
        </thead>
        <tbody id="tabla"></tbody>
    </table>

    <script>
        const API = 'http://localhost:3454';

        async function grabarAlumno() {
            const apellidos = document.getElementById('apellidos').value;
            const nombres = document.getElementById('nombres').value;
            const dni = document.getElementById('dni').value;

            const res = await fetch(`${API}/grabaAlumnos`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ apellidos, nombres, dni })
            });

            const data = await res.json();
            document.getElementById('mensaje').textContent = data.mensaje;
            if (data.resultado === 1) consultarAlumnos();
        }

        async function consultarAlumnos() {
            const res = await fetch(`${API}/consultarAlumnos`);
            const alumnos = await res.json();
            const tbody = document.getElementById('tabla');
            tbody.innerHTML = '';
            alumnos.forEach(a => {
                tbody.innerHTML += `<tr>
                    <td>${a.id}</td>
                    <td>${a.apellidos}</td>
                    <td>${a.nombres}</td>
                    <td>${a.dni}</td>
                </tr>`;
            });
        }

        consultarAlumnos();
    </script>
</body>
</html>
```

---

## Diferencias respecto a la versión Python/Flask

| Aspecto | Python/Flask | Node.js/Express |
|---|---|---|
| Runtime | python3 | node |
| Framework | Flask | Express |
| Driver MySQL | mysql-connector-python | mysql2 |
| Instalación paquetes | pip3 install | npm install |
| Archivo principal | app.py | app.js |
| Directorio del proyecto | /home/fabricio1404/ | /home/fabricio1404/app-alumnos/ |

---

## Evidencia — Sistema Funcionando

> <img width="820" height="510" alt="image" src="https://github.com/user-attachments/assets/691d20c1-cb9b-47ee-87f8-873ed1a84c1e" />


<!-- Reemplazá esta línea con tu captura: ![Sistema funcionando](./captura.png) -->

(./captura.png)

**Acceso:** `http://localhost:8080/Sistema`
