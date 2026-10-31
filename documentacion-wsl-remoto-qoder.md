# Conexión remota a WSL desde Qoder IDE

**Documentación técnica y guía paso a paso**

| Campo | Valor |
|---|---|
| Fecha | 2026-10-31 |
| Equipo auditado | Windows, usuario `tutua` |
| Distro WSL | Ubuntu 26.04.1 LTS (WSL 2) |
| Producto objetivo | Qoder IDE (fork de VS Code) |
| Estado | Procedimiento documentado; Qoder IDE no está instalado actualmente en el equipo |
| Autor | Qoder (asistente), a partir de la auditoría del entorno |

---

## 1. Objetivo

Documentar la auditoría realizada en el equipo y entregar el procedimiento completo para trabajar con Qoder IDE conectado de forma remota a una distribución de WSL 2, incluyendo requisitos, verificación, ajustes recomendados y solución de problemas.

---

## 2. Resumen ejecutivo

1. WSL 2 está correctamente instalado y operativo con Ubuntu 26.04.1 LTS.
2. Dentro de Ubuntu ya existe `~/.qoder-server`, lo que demuestra que **ya hubo al menos una conexión remota exitosa** desde Qoder IDE (servidor iniciado a las 03:34 del 2026-10-31).
3. En Windows **no está instalado Qoder IDE**: la carpeta `C:\Users\tutua\AppData\Local\Programs\Qoder IDE` está vacía y no hay procesos ni accesos directos del IDE.
4. Lo que sí está instalado y en ejecución es **Qoder 0.5.1**, la aplicación de escritorio nueva (producto distinto, orientado a agentes). Su paquete no contiene soporte de conexión WSL.
5. Por tanto, para reproducir la conexión remota hay que instalar/abrir **Qoder IDE** e instalar la extensión **Open Remote - WSL** (jeanp413), ya que la extensión oficial de Microsoft no es utilizable en forks de VS Code.

---

## 3. Auditoría del entorno

### 3.1 Sistema y WSL

Comandos ejecutados y resultado observado:

```powershell
wsl -l -v
```

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

```powershell
wsl --status
```

```text
Distribución predeterminada: Ubuntu
Versión predeterminada: 2
```

Interior de la distro:

```bash
uname -r
# 6.18.54.1-microsoft-standard-WSL2

cat /etc/os-release | head -2
# PRETTY_NAME="Ubuntu 26.04.1 LTS"
# NAME="Ubuntu"

echo $HOME
# /home/alvaro
```

### 3.2 Servidor remoto de Qoder en WSL

```bash
ls -la ~/.qoder-server
```

Contenido observado:

```text
.1716667cb46594e4c08fc5c502906988efacfc86.log
.1716667cb46594e4c08fc5c502906988efacfc86.pid
.1716667cb46594e4c08fc5c502906988efacfc86.token
.download-cache/
bin/
data/
extensions/
```

Extensiones instaladas en el lado Linux (`~/.qoder-server/extensions`):

- `detachhead.basedpyright-1.31.6`
- `ms-ceintl.vscode-language-pack-es-1.106.2025111209`
- `ms-python.autopep8-2025.2.0`
- `ms-python.debugpy-2025.14.0-linux-x64`
- `ms-python.python-2025.16.0-linux-x64`
- `ms-python.vscode-python-envs-1.8.0-linux-x64`

El log del servidor muestra el arranque del *extension host* en Linux y el mensaje `Got delay-shutdown request while in shutdown timeout, delaying`, asociado al bug conocido de reconexión descrito en la sección 9.

### 3.3 Instalaciones de Qoder en Windows

| Elemento | Ruta | Estado observado |
|---|---|---|
| Qoder 0.5.1 (app de escritorio) | `C:\Users\tutua\AppData\Local\Programs\Qoder\Qoder.exe` | Instalado y en ejecución |
| Qoder IDE | `C:\Users\tutua\AppData\Local\Programs\Qoder IDE` | Carpeta vacía, sin ejecutable |
| Datos de usuario del IDE | `C:\Users\tutua\.qoder` | Presentes (`extensions`, `argv.json`, `settings.json`, `entry`, etc.) |
| Servidor remoto en WSL | `/home/alvaro/.qoder-server` | Presente y usado |

Comprobaciones adicionales:

- El registro de Windows solo lista `Qoder 0.5.1`.
- No hay accesos directos de Qoder IDE en el menú Inicio.
- No hay procesos `Qoder IDE.exe` en ejecución.
- El bundle `app.asar` de Qoder 0.5.1 no contiene referencias a `qoder-server`, `wsl.exe` ni `remoteAuthority`; es decir, la app de escritorio no implementa el modo remoto WSL.

### 3.4 Extensiones del lado Windows

En `C:\Users\tutua\.qoder\extensions` figuran las mismas extensiones de Python/idioma, en variante Windows, pero **no** `open-remote-wsl`. La extensión de WSL deberá instalarse al abrir Qoder IDE.

---

## 4. Aclaración de productos

Es frecuente confundir ambos nombres:

| Producto | Qué es | ¿Soporta WSL remoto? |
|---|---|---|
| **Qoder IDE** | IDE basado en VS Code (kernel 1.106.x). Desde la versión 1.25.1 se instala como `Qoder IDE` con ejecutable `Qoder IDE.exe`. Última versión documentada: 1.37.0 (2026-10-27). | Sí, desde la versión 0.1.17; el modo Quest lo soporta desde 1.0.1. |
| **Qoder 0.5.1 (app de escritorio)** | Aplicación nueva orientada a agentes, con ventana de chat y herramientas propias. | No implementa el servidor remoto de Qoder IDE. |

Histórico relevante: en 1.24.1 el producto "Qoder Desktop" pasó a llamarse "Qoder IDE"; en 1.25.1 se actualizaron carpeta y ejecutable. En 2026 apareció además una app de escritorio nueva, que es la que está instalada en este equipo.

---

## 5. Requisitos

- Windows 10 (21H1 o superior) o Windows 11.
- WSL 2 con al menos una distribución instalada. En este equipo: Ubuntu 26.04.1 LTS.
- Qoder IDE 1.30.1 o superior (recomendado 1.37.0 por las mejoras de estabilidad WSL).
- `curl` o `wget` dentro de la distro (el servidor remoto los usa para descargarse).
- Conexión a Internet para la primera instalación del servidor y de extensiones.

---

## 6. Procedimiento paso a paso

### Paso 1. Actualizar y verificar WSL

En PowerShell (administrador):

```powershell
wsl --update
wsl --set-default-version 2
wsl -l -v
```

Requisito: la distro debe mostrar `VERSION 2`. En este equipo ya se cumple.

### Paso 2. Preparar utilidades dentro de Ubuntu

```powershell
wsl -d Ubuntu -e sudo apt update
wsl -d Ubuntu -e sudo apt install -y curl wget git
```

### Paso 3. Instalar Qoder IDE

1. Descargar Qoder IDE desde el sitio oficial de Qoder.
2. Instalar y abrir.
3. Verificar en **Ayuda → Acerca de** que la versión sea 1.30.1 o superior.
4. Ruta esperada del ejecutable: `C:\Users\tutua\AppData\Local\Programs\Qoder IDE\Qoder IDE.exe`.

> En este equipo este paso está pendiente: la carpeta existe pero está vacía.

### Paso 4. Instalar la extensión WSL

La extensión oficial *Remote - WSL* de Microsoft está licenciada solo para Visual Studio Code y no funciona en forks. Se usa la implementación libre **Open Remote - WSL** de jeanp413.

Opción A, desde el marketplace:

1. `Ctrl+Shift+X`.
2. Buscar **Open Remote - WSL**.
3. Instalar.

Opción B, desde VSIX:

1. Descargar el VSIX desde Open VSX:
   `https://open-vsx.org/extension/jeanp413/open-remote-wsl`
   (última versión publicada: 0.0.5, 2025-04-06).
2. `Ctrl+Shift+P` → **Extensions: Install from VSIX...** → seleccionar el archivo.

Si la extensión indica que necesita *proposed API*:

1. `Ctrl+Shift+P` → **Preferences: Configure Runtime Arguments**.
2. Editar `C:\Users\tutua\.qoder\argv.json` y añadir:

```json
"enable-proposed-api": ["jeanp413.open-remote-wsl"]
```

3. Reiniciar Qoder IDE.

### Paso 5. Conectar con WSL

1. Hacer clic en el icono remoto `><` de la esquina inferior izquierda, o abrir la paleta con `Ctrl+Shift+P`.
2. Escribir `WSL`.
3. Elegir **WSL: Connect to WSL using Distro...** y seleccionar **Ubuntu**.
   - Alternativas: **WSL: Connect to WSL** (distro predeterminada) o **WSL: Connect to WSL in New Window**.
4. La primera vez se descarga e instala el servidor en `/home/alvaro/.qoder-server`; en este equipo ya existe y la reconexión debe ser rápida.

Comandos disponibles de la extensión (verificados en su `package.json`):

- `WSL: Connect to WSL`
- `WSL: Connect to WSL in New Window`
- `WSL: Connect to WSL using Distro...`
- `WSL: Connect to WSL using Distro in New Window...`
- `WSL: Show Log`

### Paso 6. Abrir el proyecto en Linux

1. **Archivo → Abrir carpeta**.
2. Usar una ruta nativa de Linux, por ejemplo `/home/alvaro/proyectos/miapp`.

Recomendaciones:

- No trabajar en `/mnt/c/...`: el rendimiento de E/S es peor y los permisos pueden dar problemas.
- Si el proyecto está en Windows, moverlo a `\\wsl$\Ubuntu\home\alvaro\` desde el Explorador de Windows.

### Paso 7. Verificar la conexión

| Comprobación | Resultado esperado |
|---|---|
| Indicador inferior izquierdo | `WSL: Ubuntu` en verde |
| Terminal integrada (``Ctrl+` ``) | `uname -a` responde como Linux; `pwd` muestra la ruta Linux |
| Extensiones | Las de lenguaje muestran **Install in WSL: Ubuntu** |
| Servidor | Existe `/home/alvaro/.qoder-server` y su log se actualiza |
| Agente/Quest | Funciona dentro del espacio de trabajo Linux |

Instalar en el lado remoto lo que deba ejecutarse en Linux (Python, linters, depuradores). Las extensiones de interfaz pueden quedarse en Windows.

### Paso 8. Ajustes recomendados

Git dentro de Linux:

```bash
git config --global core.autocrlf input
git config --global user.name "Tu Nombre"
git config --global user.email "tu@correo"
```

Límite de watchers si el proyecto es grande:

```bash
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Recursos de WSL (opcional), en `C:\Users\tutua\.wslconfig`:

```ini
[wsl2]
memory=8GB
processors=4
```

---

## 7. Checklist de aceptación

- [ ] `wsl -l -v` muestra Ubuntu en versión 2.
- [ ] `curl` y `wget` instalados en la distro.
- [ ] Qoder IDE 1.30.1+ instalado y abierto.
- [ ] Extensión Open Remote - WSL instalada.
- [ ] Conexión realizada con `WSL: Connect to WSL using Distro...`.
- [ ] Indicador `WSL: Ubuntu` visible.
- [ ] Proyecto abierto en `/home/alvaro/...`.
- [ ] Terminal integrada ejecuta comandos Linux.
- [ ] Extensiones de lenguaje instaladas en el lado remoto.
- [ ] Servidor en `~/.qoder-server` operativo y log sin errores repetidos.

---

## 8. Registro de la auditoría (qué se hizo)

| Acción | Herramienta | Resultado |
|---|---|---|
| Listar distros y estado de WSL | `wsl -l -v`, `wsl --status` | Ubuntu 26.04, WSL 2, predeterminada, en ejecución |
| Consultar kernel, distro y usuario | `uname -r`, `os-release`, `$HOME` | Kernel 6.18.54.1 WSL2; usuario `alvaro` |
| Inspeccionar servidor remoto | `ls ~/.qoder-server` | Servidor, logs, bin, data y extensiones presentes |
| Revisar extensiones remotas | `ls ~/.qoder-server/extensions` | Python, debugpy, autopep8, basedpyright, language pack ES |
| Buscar instalación del IDE | Registro, menú Inicio, `Programs`, procesos | Qoder 0.5.1 instalado; carpeta de Qoder IDE vacía |
| Analizar producto instalado | `product.json`, `app.asar` | `productId: qoder`; sin soporte WSL remoto |
| Revisar datos del IDE | `C:\Users\tutua\.qoder` | `argv.json` (locale es), extensiones, configuración |
| Verificar extensión WSL | Open VSX y `package.json` del repo | Open Remote - WSL 0.0.5; comandos y requisitos |
| Contrastar versiones y soporte | Notas de versión de Qoder | WSL desde 0.1.17; estabilidad en 1.30.1; última 1.37.0 |

Limitaciones de la auditoría:

- No se pudo abrir Qoder IDE para validar la interfaz, porque no está instalado.
- Los nombres exactos de menús pueden variar ligeramente entre versiones del IDE.
- No se modificó ninguna configuración del sistema ni de WSL; todas las comprobaciones fueron de solo lectura.

---

## 9. Solución de problemas

### 9.1 Desconexiones en bucle o cada 60 segundos

Síntoma: el servidor arranca en un puerto nuevo y el cliente intenta reconectar con un token anterior; el log muestra `Unknown reconnection token (never seen)` y `Got delay-shutdown request while in shutdown timeout`.

Pasos:

```powershell
wsl --shutdown
```

Y, si persiste, eliminar el servidor para forzar reinstalación:

```powershell
wsl -d Ubuntu -e rm -rf ~/.qoder-server
```

Advertencia: se pierden las extensiones instaladas en el lado remoto; se reinstalan después.

Recomendación: actualizar Qoder IDE a 1.30.1 o superior, versión que mejoró velocidad de arranque y estabilidad de WSL.

### 9.2 Falla la descarga del servidor

Síntoma: `Couldn't install Qoder server on remote server, install script returned non-zero exit status`.

Comprobaciones:

```powershell
wsl -d Ubuntu -e curl -I https://qoder.com
wsl -d Ubuntu -e which curl wget
```

Si hay proxy corporativo, configurarlo también dentro de WSL. Verificar espacio en disco con `df -h`.

### 9.3 La extensión no aparece o no conecta

- Confirmar que es **Open Remote - WSL** (jeanp413), no la de Microsoft.
- Instalar el VSIX manualmente si el marketplace del fork no la muestra.
- Añadir `enable-proposed-api` en `argv.json` y reiniciar.
- Revisar **WSL: Show Log** y `~/.qoder-server/*.log`.

### 9.4 Lentitud o errores de archivos

- Mover el proyecto a `/home/alvaro/...`.
- Aumentar `fs.inotify.max_user_watches`.
- Evitar editar archivos desde Windows y Linux a la vez.

### 9.5 Plan B: Remote-SSH hacia WSL

Si el modo WSL no resulta estable:

1. Instalar y configurar OpenSSH Server en Ubuntu:

```bash
sudo apt install -y openssh-server
sudo sed -i 's/^#\?Port .*/Port 2222/' /etc/ssh/sshd_config
sudo service ssh restart
```

2. En Qoder IDE instalar **Remote - SSH**.
3. Conectar a `usuario@localhost:2222`.

Referencia detallada: guía de Aliyun sobre WSL2 + Qoder con Remote-SSH.

---

## 10. Referencias

- Qoder IDE Release Notes: https://docs.qoder.com/release-notes/desktop
- Open Remote - WSL (GitHub): https://github.com/jeanp413/open-remote-wsl
- Open Remote - WSL (Open VSX): https://open-vsx.org/extension/jeanp413/open-remote-wsl
- Foro Qoder, desconexiones WSL Remote: https://forum.qoder.com/t/windows-wsl-wsl-vscode/11391
- Foro Qoder, fallo de descarga del servidor: https://forum.qoder.com/t/wsl/2907
- VS Code + WSL (Microsoft): https://learn.microsoft.com/es-es/windows/wsl/tutorials/wsl-vscode
- Qoder con WSL2 vía Remote-SSH (Aliyun): https://developer.aliyun.com/article/1755879

---

## 11. Anexo: comandos útiles

```powershell
# Estado de WSL
wsl -l -v
wsl --status
wsl --update
wsl --shutdown

# Entrar a la distro
wsl -d Ubuntu

# Ejecutar un comando sin entrar
wsl -d Ubuntu -e uname -a
wsl -d Ubuntu -e df -h
```

```bash
# Dentro de Ubuntu
ls -la ~/.qoder-server
tail -f ~/.qoder-server/*.log
sudo apt install -y curl wget git
df -h
```

```text
Rutas clave:
C:\Users\tutua\.qoder\argv.json          Configuración de arranque del IDE
C:\Users\tutua\.qoder\settings.json      Ajustes de usuario del IDE
C:\Users\tutua\AppData\Local\Programs\Qoder IDE\   Instalación del IDE
\\wsl$\Ubuntu\home\alvaro                Acceso al sistema de archivos Linux
/home/alvaro/.qoder-server               Servidor remoto y extensiones Linux
```
