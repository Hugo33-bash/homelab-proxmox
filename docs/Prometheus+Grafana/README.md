# Manual de instalación y configuración de Prometheus + Grafana

En este manual montamos un sistema de monitorización completo sobre Linux: **Prometheus** recoge las métricas de las máquinas, **node_exporter** es el agente que las expone y **Grafana** las muestra en gráficas.

```text
Máquinas (node_exporter :9100)  ──►  Prometheus :9090  ──►  Grafana :3000
```

| Componente    | Qué hace                                              | Puerto |
| ------------- | ----------------------------------------------------- | ------ |
| Prometheus    | Recoge y almacena las métricas                        | 9090   |
| node_exporter | Agente que expone CPU, RAM, disco y red de una máquina | 9100   |
| Grafana       | Dibuja las métricas en paneles y gráficas             | 3000   |

> **Antes de empezar:** los comandos asumen un Linux con systemd y que trabajamos como `root` (si no, hay que anteponer `sudo`). En todo el manual el servidor de monitorización tiene la IP `192.168.1.69`: sustitúyela por la de tu máquina.

## Índice

1. [Instalación y configuración de Prometheus](#1-instalación-y-configuración-de-prometheus)
2. [Instalación y configuración de node_exporter](#2-instalación-y-configuración-de-node_exporter)
3. [Instalación y configuración de Grafana](#3-instalación-y-configuración-de-grafana)
4. [node_exporter en el resto de máquinas virtuales](#4-node_exporter-en-el-resto-de-máquinas-virtuales)
5. [Problemas habituales](#5-problemas-habituales)

---

## 1. Instalación y configuración de Prometheus

### 1.1 Crear el usuario de servicio

Creamos un usuario `prometheus` sin acceso por terminal. Con él se ejecutará el servicio, de modo que si alguien lo comprometiera no tendría privilegios sobre el resto del sistema.

```bash
useradd --system --no-create-home --shell /bin/false prometheus
```

> **Nota:** `--system` crea un usuario de sistema (UID bajo). Es importante para evitar conflictos si más adelante se instala algún paquete con `apt` que intente crear ese mismo usuario.

### 1.2 Crear los directorios y asignar permisos

Necesitamos un directorio para la configuración y otro para los datos. Solo el de datos pertenece al usuario `prometheus`, porque es donde necesita escribir.

```bash
mkdir /etc/prometheus
mkdir /var/lib/prometheus
chown -R prometheus:prometheus /var/lib/prometheus/
```

### 1.3 Descargar e instalar los binarios

Descargamos el archivo `.tar.gz` desde la [página de releases de Prometheus](https://github.com/prometheus/prometheus/releases). Si eliges otra versión, cambia el nombre del archivo en los comandos.

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.42.0/prometheus-2.42.0.linux-amd64.tar.gz
```

Descomprimimos y copiamos a `/usr/local/bin` los binarios `prometheus` (el servidor) y `promtool` (herramienta para validar la configuración). También copiamos las carpetas de consolas, a las que hace referencia el servicio del paso 1.5.

```bash
tar -zxpvf prometheus-2.42.0.linux-amd64.tar.gz
cd prometheus-2.42.0.linux-amd64/
cp prometheus promtool /usr/local/bin/
cp -r consoles console_libraries /etc/prometheus/
```

### 1.4 Crear el archivo de configuración

```bash
touch /etc/prometheus/prometheus.yml
chown -R prometheus:prometheus /etc/prometheus/prometheus.yml
```

Editamos el archivo (`vi /etc/prometheus/prometheus.yml`) y añadimos esta configuración:

```yaml
# Configuración global
global:
  scrape_interval: 15s       # Cada cuánto se recogen las métricas
  evaluation_interval: 15s   # Cada cuánto se evalúan las reglas
  scrape_timeout: 15s        # Tiempo máximo de espera por cada recogida

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']   # Solo monitoriza el servidor local
```

> **Importante:** el YAML es sensible a la indentación. Usa espacios (nunca tabuladores) y respeta los niveles tal como aparecen arriba.

Validamos la configuración antes de seguir:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

### 1.5 Crear el servicio systemd

Creamos el archivo del servicio (`vi /etc/systemd/system/prometheus.service`) con este contenido:

```ini
[Unit]
Description=Prometheus Time Series Collection and Processing Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --storage.tsdb.retention.time=30d \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

La opción `--storage.tsdb.retention.time=30d` hace que Prometheus conserve las métricas durante 30 días y borre las más antiguas, para que el disco no se llene.

### 1.6 Iniciar el servicio y verificar

Recargamos systemd para que lea el nuevo servicio, lo iniciamos y lo habilitamos para que arranque con el sistema:

```bash
systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus
systemctl status prometheus
```

Comprobamos que está escuchando en el puerto 9090:

```bash
ss -tpln | grep 9090
```

Ahora entramos a la interfaz web desde el navegador: `http://192.168.1.69:9090`.

En el cuadro de consulta escribimos `up` y pulsamos **Execute**. Aparecerá una fila por cada servicio monitorizado, con valor `1` si está funcionando: primero solo `prometheus` y, cuando terminemos el apartado 2, también `node_exporter`.

![Consulta «up» en la interfaz web de Prometheus mostrando los servicios prometheus y node_exporter](prometheus-up.png)

---

## 2. Instalación y configuración de node_exporter

> **Nota:** node_exporter es el agente que recopila y expone una gran cantidad de métricas de nuestro servidor Linux, como CPU, uso de memoria, sistemas de archivos y estadísticas de red. Prometheus lo consulta periódicamente.

### 2.1 Crear el usuario de servicio

```bash
useradd --system --no-create-home --shell /bin/false node_exporter
```

### 2.2 Descargar e instalar el binario

Descargamos el `.tar.gz` desde la [página de releases de node_exporter](https://github.com/prometheus/node_exporter/releases):

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
```

Descomprimimos y copiamos el binario `node_exporter` a `/usr/local/bin`:

```bash
tar -zxpvf node_exporter-1.5.0.linux-amd64.tar.gz
cd node_exporter-1.5.0.linux-amd64
cp node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### 2.3 Crear el servicio systemd

Creamos el archivo (`vi /etc/systemd/system/node_exporter.service`) con este contenido:

```ini
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

### 2.4 Iniciar el servicio y verificar

```bash
systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
systemctl status node_exporter
```

Verificamos que esté escuchando en el puerto **9100**:

```bash
ss -tlpn | grep 9100
```

### 2.5 Añadir el job a Prometheus

Para que Prometheus recoja las métricas del node_exporter, añadimos un nuevo job al archivo `/etc/prometheus/prometheus.yml`. El archivo completo queda así:

```yaml
# Configuración global
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Validamos y reiniciamos Prometheus para que cargue la nueva configuración:

```bash
promtool check config /etc/prometheus/prometheus.yml
systemctl restart prometheus.service
```

Con esto ya podemos ver las métricas desde la interfaz web (consulta `up`, o por ejemplo `node_memory_MemAvailable_bytes`).

---

## 3. Instalación y configuración de Grafana

> **Nota:** Grafana es la herramienta que nos permite visualizar en paneles (dashboards) y gráficas las métricas que recopila Prometheus. Prometheus almacena los datos y Grafana los muestra.

Esta parte está pensada para Debian y Ubuntu.

### 3.1 Añadir el repositorio oficial

Instalamos las dependencias necesarias:

```bash
apt update
apt install -y apt-transport-https wget gpg
```

Descargamos la clave GPG de Grafana y añadimos el repositorio oficial, así podremos instalarlo y actualizarlo con `apt`:

```bash
mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor > /etc/apt/keyrings/grafana.gpg
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" > /etc/apt/sources.list.d/grafana.list
```

### 3.2 Instalar Grafana

```bash
apt update
apt install -y grafana
```

### 3.3 Iniciar el servicio y verificar

La instalación crea automáticamente el usuario de servicio y el servicio systemd `grafana-server`. Lo iniciamos y lo habilitamos para que arranque con el sistema:

```bash
systemctl daemon-reload
systemctl start grafana-server
systemctl enable grafana-server
systemctl status grafana-server
```

Verificamos que esté escuchando en el puerto **3000**:

```bash
ss -tlpn | grep 3000
```

Si el equipo tiene el firewall activado, debemos permitir el acceso a ese puerto (por ejemplo, con `ufw`):

```bash
ufw allow 3000/tcp
```

### 3.4 Primer acceso

Entramos desde el navegador a `http://192.168.1.69:3000`. Las credenciales por defecto son usuario `admin` y contraseña `admin`. Al entrar por primera vez nos pedirá cambiar la contraseña, y es importante hacerlo.

### 3.5 Conectar Grafana con Prometheus

Añadimos Prometheus como origen de datos (*data source*). Desde el menú lateral vamos a **Connections → Data sources → Add data source** y elegimos **Prometheus**.

En el campo **URL** escribimos la dirección de Prometheus. Si Grafana y Prometheus están en la misma máquina:

```text
http://localhost:9090
```

Si Prometheus está en otra máquina, usamos su IP, por ejemplo `http://192.168.1.69:9090`.

Dejamos el resto de opciones por defecto, bajamos hasta el final y pulsamos **Save & test**. Si todo es correcto, aparecerá un mensaje en verde indicando que se ha consultado la API de Prometheus correctamente.

### 3.6 Importar un dashboard

Importamos un panel ya preparado para visualizar las métricas de node_exporter. Vamos a **Dashboards → New → Import**, escribimos el ID **1860** (*Node Exporter Full*), pulsamos **Load**, seleccionamos nuestro origen de datos de Prometheus y pulsamos **Import**.

Con esto ya podemos ver el uso de CPU, memoria, disco y red del servidor. En la parte superior del panel hay un desplegable llamado **instance** para elegir de qué máquina queremos ver las métricas: cuando añadamos más servidores en el `prometheus.yml` aparecerán ahí.

### 3.7 Cambiar el puerto (opcional)

Si en algún momento necesitamos cambiar el puerto o la configuración de Grafana, se hace en el archivo `/etc/grafana/grafana.ini` (sección `[server]`, parámetro `http_port`) y después reiniciamos el servicio:

```bash
systemctl restart grafana-server
```

---

## 4. node_exporter en el resto de máquinas virtuales

> **Nota:** para centralizar las métricas solo hay que instalar el agente node_exporter en cada máquina virtual. Prometheus y Grafana se quedan en el servidor central: Prometheus recoge los datos de todas las máquinas y Grafana los muestra en las gráficas.

> **Atención:** no instales el paquete de `apt` en la máquina donde ya hiciste la instalación manual del apartado 2. Ambos usarían el puerto 9100 y el mismo nombre de servicio, y se estorbarían entre sí. Elige un único método por máquina.

### 4.1 Instalar el agente

Entramos por terminal en cada una de las máquinas virtuales que queremos monitorizar. En Debian y Ubuntu instalamos el paquete desde los repositorios, que ya crea el usuario y el servicio, y lo habilitamos:

```bash
apt update
apt install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

Para otras distribuciones, seguimos los pasos del apartado 2 (instalación manual).

### 4.2 Verificar

Comprobamos que escucha en el puerto 9100 y que devuelve métricas:

```bash
ss -tlpn | grep 9100
curl -s localhost:9100/metrics | head
```

Si todo sale bien, la salida mostrará líneas como `node_cpu_seconds_total`.

### 4.3 Permitir el acceso solo desde Prometheus

Permitimos el acceso al puerto 9100 únicamente desde el servidor de Prometheus (en nuestro caso `192.168.1.69`). Así evitamos que cualquier otro equipo de la red pueda leer las métricas. En Debian y Ubuntu con `ufw`:

```bash
ufw allow from 192.168.1.69 to any port 9100 proto tcp
```

### 4.4 Añadir las máquinas a Prometheus

En el servidor de Prometheus, añadimos las máquinas nuevas al job `node_exporter` del archivo `/etc/prometheus/prometheus.yml` (cambiando las IP de ejemplo por las de nuestras máquinas):

```yaml
  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - 'localhost:9100'
          - '192.168.1.70:9100'
          - '192.168.1.71:9100'
```

Validamos y reiniciamos el servicio de Prometheus para que cargue la nueva configuración:

```bash
promtool check config /etc/prometheus/prometheus.yml
systemctl restart prometheus.service
```

### 4.5 Comprobar que Prometheus las ve

Abrimos la interfaz web de Prometheus, en **Status → Targets** (`http://192.168.1.69:9090/targets`), y comprobamos que las máquinas aparecen en estado **UP**. Si alguna sale **DOWN**, normalmente es porque el firewall bloquea el puerto 9100 o porque el servicio node_exporter no está en marcha.

### 4.6 Verlas en Grafana

Para terminar, en Grafana abrimos el dashboard **Node Exporter Full** y, en el desplegable **instance** de la parte superior, elegimos la máquina de la que queremos ver las gráficas. Pueden tardar unos 30 segundos en aparecer los primeros datos.

---

## 5. Problemas habituales

| Síntoma | Causa probable | Qué comprobar |
| ------- | -------------- | ------------- |
| Un target aparece en **DOWN** | El firewall bloquea el 9100 o node_exporter está parado | `ufw status` y `systemctl status node_exporter` en esa máquina |
| Prometheus no arranca tras editar la configuración | Error de sintaxis o de indentación en el YAML | `promtool check config /etc/prometheus/prometheus.yml` y `journalctl -u prometheus -e` |
| Un servicio falla con «permission denied» | Binario o directorios sin los permisos correctos | Revisar los `chown` de los apartados 1.2, 1.4 y 2.2 |
| **Save & test** falla en Grafana | La URL de Prometheus es incorrecta o el puerto 9090 no es accesible | Si están en máquinas distintas, usar la IP y no `localhost` |
| El dashboard aparece vacío | Rango de tiempo o `instance` mal elegidos, o aún no hay datos | Elegir «Last 15 minutes» y la máquina correcta; esperar unos 30 segundos |
