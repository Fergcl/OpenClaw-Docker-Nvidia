# 🦞 Asistente de IA autoalojado con OpenClaw, Docker y la API de NVIDIA

> Despliegue de un agente de IA personal y autoalojado (self-hosted) sobre Docker,
> usando modelos servidos por la API de NVIDIA (NIM). Proyecto de aprendizaje
> realizado durante mis estudios de **DAW (Desarrollo de Aplicaciones Web)**.

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

```
┌─────────────────────────────────────────────────────────────┐
│  Windows 11 (host)                                           │
│                                                             │
│   Navegador  ──HTTP──►  localhost:18789                      │
│                              │                              │
│   ┌──────────────────────────┼───────────────────────────┐  │
│   │  Docker (sobre WSL2)      │                           │  │
│   │                           ▼                           │  │
│   │   ┌─────────────────────────────────────────────┐    │  │
│   │   │  Contenedor: OpenClaw Gateway               │    │  │
│   │   │  - Node.js + OpenClaw (todo incluido)       │    │  │
│   │   │  - Agente con workspace (.md) y memoria     │    │  │
│   │   └──────────────────┬──────────────────────────┘    │  │
│   │                      │                                │  │
│   │     Volúmenes (-v):  │  ~/.openclaw  (config + datos) │  │
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
tareas que requieren herramientas.

**Aprendizaje:** no todos los modelos sirven para lo mismo. El "tool calling" es
una capacidad concreta que conviene verificar, no dar por hecha.

### 5. Desajuste de versiones en la imagen

**Causa:** la imagen traía la CLI y el gateway en versiones distintas, lo que
provocaba que ciertas claves de configuración (y un plugin) fueran rechazadas.

**Solución:** identificar el desajuste en los logs y trabajar con las claves que
la versión activa sí aceptaba, evitando las que no.

**Aprendizaje:** leer los logs con atención ahorra horas. El mensaje
`version mismatch` explicaba de golpe varios errores aparentemente inconexos.

---

## 🧠 Configuración del agente

El comportamiento del agente se define con archivos **Markdown** en su *workspace*,
independientes del modelo (se puede cambiar de modelo sin perder la personalidad):

| Archivo       | Función                                            |
|---------------|----------------------------------------------------|
| `SOUL.md`     | Personalidad, tono, reglas de idioma y de conducta |
| `USER.md`     | Información y preferencias del usuario              |
| `MEMORY.md`   | Memoria a largo plazo (hechos duraderos)           |
| `IDENTITY.md` | Nombre, emoji y "vibe" del agente                  |

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
- **Buenas prácticas de seguridad:** no exponer secretos (tokens, API keys) ni
  datos personales en un repositorio público.

---

## ⚠️ Nota sobre seguridad

Este repositorio contiene **únicamente documentación**. No incluye claves API,
tokens ni datos personales. La configuración real y los secretos se mantienen
fuera del control de versiones (`.gitignore`), como debe ser en cualquier proyecto.

---

## 🔗 Referencias

- OpenClaw — https://github.com/openclaw/openclaw
- Documentación oficial — https://docs.openclaw.ai
- NVIDIA NIM (modelos) — https://build.nvidia.com

---

_Proyecto de aprendizaje · Estudiante de DAW · 2026_
