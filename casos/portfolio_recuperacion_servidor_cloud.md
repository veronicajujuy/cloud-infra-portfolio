# 🧾 Recuperación de servidor en producción — Docker + MySQL + Cloud

> **Resumen ejecutivo:** Servidor cloud en producción (múltiples instancias Moodle en Docker) quedó inaccesible por saturación de disco (93%), con MySQL en loop de reinicio y acceso SSH caído. Diagnostiqué la causa raíz (logs Docker sin rotación, ~246GB acumulados), liberé 130GB sin downtime, y recuperé MySQL recreando el contenedor con el socket redirigido — **sin pérdida de datos y con el servicio restaurado en ~3 horas**. También se detectó y descartó un intento de intrusión (webshell) en los logs.

**Rol:** Administración de infraestructura / DevOps
**Entorno:** Cloud ECS · Ubuntu 18.04 · Docker · MySQL 5.7 · Moodle
**Duración del incidente:** ~3 horas
**Resultado:** Servicio restaurado al 100% sin pérdida de datos

---

## 📌 Contexto

Servidor cloud con múltiples instancias de Moodle ejecutándose en contenedores Docker.
El sistema presentó inestabilidad generalizada:

- Imposibilidad de acceso por SSH.
- Instancia en estado *Interrumpido* en consola cloud.
- Sitio principal mostrando: `Database connection failed`.
- Disco principal al 93% de uso.
- Contenedor de base de datos en loop de reinicios.

La infraestructura incluía:

- VM Ubuntu 18.04
- Docker con múltiples contenedores (Moodle, MySQL, herramientas auxiliares)
- Base de datos MySQL 5.7 en contenedor
- Volúmenes separados para base de datos, archivos Moodle y backups.

---

## 🔍 Diagnóstico Técnico

### 1️⃣ Problema de acceso SSH

Se detectaron errores:

- `Permission denied (publickey)`
- Formato inválido de clave privada
- El administrador de una de las áreas no lograba conectarse por SSH ni FTP
- Instancia en estado freezado, no se podía loguear desde el acceso interno del proveedor cloud

---

### 2️⃣ Disco al 93% — Saturación por logs Docker

Verificación:

```bash
df -h /
du -sh /var/lib/docker
docker system df
```

Hallazgo:

- `/var/lib/docker` ocupaba ~246GB.
- 52 imágenes, 79 contenedores, 45 volúmenes.
- Logs individuales de contenedores superaban cientos de MB.

Causa raíz:
No existía rotación de logs Docker → crecimiento ilimitado durante años.

---

### 3️⃣ Intento de intrusión detectado

En los logs de Moodle se identificaron requests intentando ejecutar una webshell remota.

Los intentos devolvían 404 → el exploit no tuvo éxito.

---

### 4️⃣ MySQL en loop de reinicio

El contenedor de base de datos de uno de los servicios de Moodle principales se reiniciaba indefinidamente.

Error observado:

```
Unable to setup unix socket lock file
Aborting
```

Diagnóstico:

- `/var/run` es un filesystem temporal (tmpfs).
- El llenado de disco provocó apagado abrupto.
- El contenedor MySQL quedó en estado inválido.
- El directorio `/var/run/mysqld` no se recreaba correctamente.

---

## 🛠️ Soluciones Aplicadas

### 🔹 Recuperación de acceso a VM

- Se reinició la instancia desde consola cloud y se restableció la contraseña.
- Acceso por consola VNC.

---

### 🔹 Liberación de espacio (sin downtime)

```bash
truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

Resultado:

- Uso de disco: 93% → 54%
- ~130GB liberados
- Sin eliminar contenedores ni volúmenes

---

### 🔹 Reparación de contenedor MySQL

Se recreó el contenedor redirigiendo el socket a `/tmp`:

```bash
docker stop mysql-container
docker rm mysql-container

docker run -d --name mysql-container \
  --network app_network \
  -v /ruta/datos/mysql:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=*** \
  mysql:5.7 --socket=/tmp/mysql.sock
```

Importante:

- El volumen de datos no fue modificado.
- No hubo pérdida de información.
- Servicio restaurado inmediatamente.

---

## 📊 Estado Final

| Componente   | Estado          |
|--------------|-----------------|
| Acceso SSH   | ✅ Operativo     |
| Instancia VM | ✅ En ejecución  |
| Disco raíz   | ✅ 54% de uso    |
| MySQL        | ✅ Corriendo     |
| Moodle       | ✅ Operativo     |

---

## 📚 Acciones Preventivas Recomendadas

- Configurar rotación de logs Docker:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

- Ejecutar mantenimiento periódico: `docker system prune`
- Revisar volúmenes huérfanos
- Reemplazar acceso por contraseña por nuevo keypair SSH
- Implementar monitoreo de uso de disco y alertas

---

## 🎯 Notas del Incidente

- `/var/run` es volátil → reinicios abruptos pueden afectar contenedores.
- Los logs Docker pueden saturar disco si no se limitan.
- `truncate` permite liberar espacio sin detener producción.
- `docker rm` elimina solo la configuración del contenedor, no los datos.
- Arquitecturas con múltiples volúmenes requieren monitoreo continuo.

---

## 💡 Competencias Aplicadas

- Diagnóstico en producción bajo presión.
- Análisis de logs y forense básico.
- Administración avanzada de Docker.
- Gestión de almacenamiento en entornos cloud.
- Recuperación segura de bases de datos en contenedor.
- Resolución de incidentes sin pérdida de datos.

---

**Caso real de recuperación de infraestructura cloud en entorno educativo.**