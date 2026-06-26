# 🦞 Asistente de IA autoalojado con OpenClaw, Docker y la API de NVIDIA
<br>
<img width="1917" height="924" alt="Panel con Tenacitas respondiendo" src="https://github.com/user-attachments/assets/63693a46-473d-47d9-a964-8b30bee8faeb" />
<br>
  


> Despliegue de un agente de IA personal y autoalojado (self-hosted) sobre Docker,
> usando modelos servidos por la API de NVIDIA (NIM). Proyecto de aprendizaje.

Este repositorio documenta, de principio a fin, cómo monté un asistente de IA
privado corriendo en mi propia máquina: la instalación, las decisiones técnicas,
los problemas reales que aparecieron y cómo los resolví.

---

## 📌 Qué es este proyecto

[OpenClaw](https://github.com/openclaw/openclaw) es una plataforma de código
abierto (licencia MIT) que actúa como **puerta de enlace (gateway)** entre un
canal de chat (un panel web, WhatsApp, Telegram…) y un modelo de IA. El modelo no
tiene por qué ser de pago: en este proyecto uso la **API gratuita de NVIDIA (NIM)**,
compatible con el estándar de OpenAI.

El resultado es un asistente personal que:

- Corre **aislado en un contenedor Docker**, sin ensuciar el sistema anfitrión.
- Usa modelos abiertos servidos por NVIDIA (Nemotron, GLM, etc.).
- Mantiene **memoria persistente** y una personalidad configurable.
- Es accesible desde el navegador mediante un panel web local.
- Puede **buscar en internet** mediante una herramienta integrada.

---

## 🏗️ Arquitectura

- Sin inciar Docker ni el servicio:
<img width="1915" height="1024" alt="Captura de pantalla 2026-06-26 010627" src="https://github.com/user-attachments/assets/9c604b00-49b6-4e75-a73f-5e32bf9562f0" />
<br>
<br>

- Iniciando Docker y el servicio openclaw a la vez:
<img width="1909" height="1031" alt="Captura de pantalla 2026-06-26 010712" src="https://github.com/user-attachments/assets/2d46b1b8-e6da-4e16-8f6f-e510f1762d59" />
<br>
<br>


```
┌─────────────────────────────────────────────────────────────┐
│  Windows 11 (host)                                          │
│                                                             │
│   Navegador  ──HTTP──►  localhost:18789                     │
│                              │                              │
│   ┌──────────────────────────┼───────────────────────────┐  │
│   │  Docker (sobre WSL2)      │                          │  │
│   │                           ▼                          │  │
│   │   ┌─────────────────────────────────────────────┐    │  │
│   │   │  Contenedor: OpenClaw Gateway               │    │  │
│   │   │  - Node.js + OpenClaw (todo incluido)       │    │  │
│   │   │  - Agente con workspace (.md) y memoria     │    │  │
│   │   └──────────────────┬──────────────────────────┘    │  │
│   │                      │                               │  │
│   │    Volúmenes (-v):     ~/.openclaw  (config + datos) │  │
│   └──────────────────────┼────────────────────────────────┘ │
└──────────────────────────┼──────────────────────────────────┘
                           │  HTTPS (API key)
                           ▼
              https://integrate.api.nvidia.com/v1
                 (modelos: Nemotron / GLM / …)
```

**Idea clave:** OpenClaw es un *gateway*, no una IA. No piensa: enruta. El que
genera las respuestas es el modelo (alojado en NVIDIA). OpenClaw conecta el canal
con ese modelo y le aporta memoria, herramientas y personalidad.

---

## 💾 Persistencia de datos: volúmenes Docker

Una decisión clave del despliegue: **el contenedor corre en Docker, pero los
datos viven en el host** (Windows), no dentro del contenedor.

Esto se consigue con los volúmenes (`-v`) del comando de arranque:

```bash
-v $HOME/.openclaw:/home/node/.openclaw
```

Esa línea "enchufa" la carpeta `~/.openclaw` del host dentro del contenedor, de
modo que toda la configuración, la memoria del agente y el workspace se guardan
fuera del contenedor.

**¿Por qué importa?** Los contenedores Docker son desechables por diseño. Si los
datos vivieran dentro del contenedor, al borrarlo o recrearlo se perdería todo:
configuración, memoria y personalidad del agente. Durante este proyecto borré y
recreé el contenedor varias veces (para actualizar la imagen, resolver errores,
etc.) y el agente **nunca perdió su estado**, porque los datos persistían en el
host.


| Contenedor Docker | El software en ejecución. Desechable y reemplazable. |
| Carpeta `~/.openclaw` (host) | Los datos persistentes. Config, memoria y workspace. |

**Ventajas prácticas de este enfoque:**

- **Edición sencilla:** pude editar los archivos de configuración (`openclaw.json`,
  `SOUL.md`…) directamente desde el host, sin entrar al contenedor.
- **Copia de seguridad trivial:** basta con copiar la carpeta `~/.openclaw` para
  respaldar todo el estado del agente.
- **Portabilidad:** mover el proyecto a otro equipo es tan simple como llevar esa
  carpeta y apuntar un contenedor nuevo a ella.

> Es la práctica recomendada de Docker: separar el ciclo de vida del **contenedor**
> (efímero) del de los **datos** (persistentes).

---

## 🛠️ Stack técnico

| Componente        | Tecnología                                   |
|-------------------|----------------------------------------------|
| Contenedorización | Docker (Docker Desktop sobre WSL2)           |
| Aplicación        | OpenClaw (Node.js, código abierto)           |
| Inferencia        | API de NVIDIA NIM (compatible con OpenAI)    |
| Modelos probados  | Nemotron 3 (Super/Ultra), GLM-5.1            |
| Persistencia      | Volúmenes Docker + archivos Markdown + SQLite|
| Acceso            | Panel web local (puerto 18789)               |

---

## 🚀 Instalación (resumen)

> Requisitos: Docker + Docker Compose v2, una clave API de
> [build.nvidia.com](https://build.nvidia.com) (tier gratuito).

**1. Obtener la imagen** (usando una imagen preconstruida para evitar compilar):

```bash
docker pull <imagen-openclaw>:latest
```

**2. Pasar la clave de NVIDIA como variable de entorno:**

```bash
export NVIDIA_API_KEY="nvapi-..."   # en PowerShell: $env:NVIDIA_API_KEY="nvapi-..."
```

**3. Ejecutar el asistente de configuración (onboarding):**

```bash
docker run -it --rm \
  -e NVIDIA_API_KEY=$NVIDIA_API_KEY \
  -v $HOME/.openclaw:/home/node/.openclaw \
  <imagen-openclaw>:latest onboard
```

**4. Arrancar el gateway de forma persistente:**

```bash
docker run -d \
  --name openclaw \
  --restart unless-stopped \
  -e NVIDIA_API_KEY=$NVIDIA_API_KEY \
  -v $HOME/.openclaw:/home/node/.openclaw \
  -p 18789:18789 \
  <imagen-openclaw>:latest gateway
```

**5. Acceder al panel** en `http://localhost:18789` (con el token generado en el
onboarding).

> Explicación de las opciones del comando: `-d` lo deja en segundo plano;
> `--restart unless-stopped` lo relanza tras reiniciar el equipo; `-v` monta una
> carpeta del host para que la configuración persista; `-p` publica el puerto del
> gateway hacia el host. Se usa `gateway` (primer plano) y no `gateway start`
> (segundo plano), porque si el proceso principal del contenedor pasa a segundo
> plano, Docker da por terminado el contenedor y lo apaga.

---

## 🐞 Problemas reales y cómo los resolví

La parte más instructiva del proyecto. Cada uno me obligó a leer logs, entender la
causa y aplicar una solución concreta.

### 1. El panel web no cargaba (`ERR_EMPTY_RESPONSE`)

**Causa:** dentro del contenedor, "esta máquina" (loopback / 127.0.0.1) es el
propio contenedor, no el host. El gateway escuchaba solo en loopback interno, así
que el navegador del host no llegaba, pese a estar el puerto publicado.

**Solución:** hacer que el gateway escuche en todas las interfaces internas:

```bash
docker exec openclaw openclaw config set gateway.bind lan
docker restart openclaw
```

**Aprendizaje:** entender la diferencia entre el `localhost` del host y el del
contenedor es fundamental en redes con Docker.

### 2. Configuración inválida que bloqueaba el arranque

**Causa:** el onboarding guardó un proveedor de búsqueda con un nombre que el
validador no reconocía (`parallel-free`), dejando toda la config en estado
inválido. Y al estar inválida, los propios comandos `config set` se negaban a
modificarla: un círculo vicioso.

**Solución:** editar el archivo `openclaw.json` a mano para eliminar la clave
problemática, validar y reiniciar.

**Aprendizaje:** cuando una herramienta se niega a auto-repararse, hay que bajar al
archivo de configuración y entender su estructura (aquí, JSON).

### 3. Timeouts al usar un modelo demasiado grande

**Causa:** el modelo más potente (Nemotron Ultra, 550B) era demasiado lento en el
tier gratuito y no emitía respuesta antes de que saltara el *watchdog* de
inactividad (`AbortError`).

**Solución:** cambiar a un modelo más ágil (Nemotron Super / GLM-5.1).

**Aprendizaje:** "el modelo más grande" no siempre es la mejor decisión. Hay que
equilibrar **potencia, velocidad y fiabilidad** según el caso de uso real.

### 4. Un modelo que no sabía usar herramientas

**Causa:** GLM-5.1 daba muy buenas respuestas de chat, pero **fallaba al llamar a
herramientas** (búsqueda web): generaba un formato que OpenClaw no podía
interpretar (`reason=format`, respuesta vacía).

**Solución:** usar Nemotron, que maneja correctamente el *tool calling*, para las
tareas que requieren herramientas. El cambio se hace en `openclaw.json`,
fijando el modelo primario:

```json
{
  "agents": {
    "defaults": {
      "workspace": "/home/node/.openclaw/workspace",
      "models": {
        "nvidia/nemotron-3-ultra-550b-a55b": {},
        "nvidia/z-ai/glm-5.1": {},
        "nvidia/moonshotai/kimi-k2.5": {},
        "nvidia/nemotron-3-super-120b-a12b": {}
      },
      "model": {
        "primary": "nvidia/nemotron-3-super-120b-a12b"
      }
    }
  },
```

**Aprendizaje:** no todos los modelos sirven para lo mismo. El "tool calling" es
una capacidad concreta que conviene verificar, no dar por hecha.


### 5. Desajuste de versiones en la imagen

**Causa:** la imagen traía la CLI y el gateway en versiones distintas, lo que
provocaba que ciertas claves de configuración (y un plugin) fueran rechazadas.
- El plugin de búsqueda Parallel requería una versión más nueva (>=2026.6.10) que la que tenía la CLI, así que se rechazaba y dejaba la config inválida (parallel-free).
- La clave del timeout (agents.defaults.llm.idleTimeoutSeconds) que intentamos añadir era rechazada por el validador viejo (Invalid input), aunque es una clave válida en versiones más nuevas.

**Solución:** Rodeamos el problema para no generar más problemas.
- Abandonamos el plugin Parallel para la búsqueda web y usamos DuckDuckGo, que viene integrado y no depende de ese plugin problemático. (config set tools.web.search.provider duckduckgo)
- Renunciamos a subir el timeout vía esa clave, porque el validador la rechazaba. En su lugar, resolvimos el problema de fondo (los timeouts) cambiando a un modelo más rápido (Nemotron Super), que no necesitaba más tiempo de espera.
- Limpiamos a mano las entradas de config (openclaw.json) que el desajuste había dejado rotas.

**Aprendizaje:** leer los logs con atención ahorra horas. El mensaje
`version mismatch` explicaba de golpe varios errores aparentemente inconexos.

---

## 🧠 Configuración del agente

Toda la configuración y el estado del agente viven en la carpeta `.openclaw`,
que (gracias a los volúmenes de Docker) se guarda en el host (mi user en windows),
no dentro del contenedor. Su estructura es la siguiente:

```text
.openclaw/
├── openclaw.json          # Configuración técnica (modelo, gateway, herramientas)
│
└── workspace/             # El "cerebro" del agente (personalidad y memoria). OTRO WORKSPACE (EJ: AGENTE2) ES OTRO AGENTE
    ├── SOUL.md            # Personalidad, tono y reglas de conducta e idioma
    ├── IDENTITY.md        # Nombre, emoji y "vibe" del agente
    ├── USER.md            # Información y preferencias del usuario
    ├── MEMORY.md          # Memoria a largo plazo (hechos duraderos)
    ├── AGENTS.md          # Instrucciones operativas (reglas de funcionamiento)
    ├── TOOLS.md           # Notas sobre el entorno y las herramientas
    └── memory/            # Diario por días (notas y resúmenes de cada sesión)
        └── AAAA-MM-DD.md
```

**Dos niveles bien diferenciados:**

- **`openclaw.json`** → la configuración *técnica*: qué modelo se usa, en qué
  puerto escucha el gateway, qué herramientas están activas, etc.
- **`workspace/`** → la *identidad* del agente: quién es, cómo habla y qué
  recuerda. Son archivos Markdown **independientes del modelo**: se puede cambiar
  de modelo (de GLM a Nemotron, por ejemplo) sin que el agente pierda su
  personalidad ni su memoria.

| Archivo | Función |
|---------|---------|
| `openclaw.json` | Configuración técnica: modelo, gateway, herramientas |
| `workspace/SOUL.md` | Personalidad, tono, reglas de idioma y de conducta |
| `workspace/IDENTITY.md` | Nombre, emoji y "vibe" del agente |
| `workspace/USER.md` | Información y preferencias del usuario |
| `workspace/MEMORY.md` | Memoria a largo plazo (hechos duraderos) |
| `workspace/AGENTS.md` | Instrucciones operativas (reglas de funcionamiento) |
| `workspace/TOOLS.md` | Notas sobre el entorno y las herramientas |
| `workspace/memory/` | Diario por días con notas y resúmenes de cada sesión |


---

## 📚 Lo que aprendí con este proyecto

- **Docker en profundidad:** imágenes, contenedores, volúmenes, publicación de
  puertos, persistencia de datos y diferencias entre host y contenedor.
- **WSL2** como base de Docker en Windows.
- **APIs compatibles con OpenAI** y cómo conectar un proveedor de modelos (NVIDIA).
- **Lectura y diagnóstico de logs** para encontrar la causa raíz de un fallo.
- **Gestión de configuración** en JSON y resolución de estados inválidos.
- **Criterio para elegir modelos** según velocidad, fiabilidad y capacidades
  (como el *tool calling*), no solo por su tamaño.

---

## ⚠️ Nota sobre seguridad

Este repositorio contiene **únicamente documentación**. No incluye claves API,
tokens ni datos personales.

---

## 🔗 Referencias

- OpenClaw — https://github.com/openclaw/openclaw
- Documentación oficial — https://docs.openclaw.ai
- NVIDIA NIM (modelos) — https://build.nvidia.com

---

_Proyecto de aprendizaje · Estudiante de DAW · 2026_
