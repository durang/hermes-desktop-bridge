# Setup From Zero — Guía Súper Clara

> Diseñada para personas que NUNCA han hecho esto antes. Si te pierdes en
> algún paso, copia el contenido del paso completo y pégaselo a
> [Claude Code](https://claude.ai/code). Te va a guiar línea por línea.

## ¿Qué vas a lograr al terminar?

Tu Mac (laptop o iMac) se va a conectar de forma privada y segura con un
servidor en la nube (EC2 / Linux box) usando Tailscale. Después podrás:

- Mandar comandos desde tu cel/lentes/Telegram a tu Mac sin abrirla.
- Preguntar "qué archivos hay en mi Desktop" desde cualquier lugar.
- Pedirle que lea tu blog, edite código, abra un repo, todo por voz/chat.

**No vas a abrir puertos públicos. No vas a exponer tu Mac a internet abierta.**
Todo viaja por Tailscale, una VPN privada gratis.

---

## Antes de empezar

Necesitas:
1. Una Mac (macOS 12+).
2. Una cuenta de Tailscale (gratis, https://tailscale.com).
3. Hermes ya instalado en tu Mac. Si no lo tienes, instala Hermes primero.
4. Un EC2 / Linux box donde corre tu Hermes gateway (Sergio's pattern: EC2
   con `hermes` corriendo como systemd service).
5. 15-20 minutos.

Si NO sabes qué es alguna de estas cosas, pásale este archivo a Claude Code
y dile: *"explícame cada punto antes de empezar"*. Te va a aclarar.

---

## Paso 1 — Verificar Tailscale en tu Mac

### ¿Qué hace este paso?

Tailscale es la "VPN privada" que conecta tu Mac con tu servidor sin
abrir puertos. Es como tener una red local privada entre tus máquinas,
aunque estén en redes distintas.

### Lo que tienes que hacer

1. Abre tu Mac.
2. Verifica que Tailscale está instalado. Abre Terminal (Cmd+Espacio,
   escribe "Terminal", Enter) y corre:

   ```bash
   tailscale status
   ```

3. Esperas ver una lista de tus máquinas. Si dice "command not found":
   instala Tailscale desde https://tailscale.com/download/mac
4. Anota tu **Tailscale IP**. Aparece en la primera línea de `tailscale status`,
   algo como `100.x.y.z`. La vas a usar en el Paso 3.

### Si te trabas

Copia este paso completo y pégaselo a Claude Code. Dile:
> "Me trabé en el Paso 1 del SETUP-FROM-ZERO. Mi Tailscale dice esto: <pega
> el output>. Ayúdame a entender qué falta."

---

## Paso 2 — Generar tu "Bearer key" privada

### ¿Qué es esto?

Es una contraseña aleatoria larga. La usa tu servidor EC2 para identificarse
con tu Mac cuando se quiera conectar. Sin esta key, nadie más puede hacer
requests a tu Mac (ni siquiera otros de tu tailnet).

### Lo que tienes que hacer

En la Terminal de tu Mac:

```bash
# Genera una key aleatoria de 32 caracteres hex (256 bits — más que suficiente)
openssl rand -hex 32
```

**Cópiala** y guárdala en algún lugar seguro temporal (notas del iPhone,
1Password, lo que uses). Se ve algo así:

```
4a7c9e2b8d6f1e3a5b9d2c4e7f1a3b5d8e2f4a6c9b1d3e5f7a8c2b4d6e1f3a5b
```

⚠️ Esta key NO se debe compartir con nadie. Si la pierdes, simplemente
genera otra y actualiza los lugares que la usen.

---

## Paso 3 — Configurar y arrancar el Hermes en tu Mac

### ¿Qué hace este paso?

Le dice a Hermes en tu Mac: "expón una API privada en el puerto 8642 para
que tu Mac pueda recibir requests del EC2".

### Lo que tienes que hacer

1. Abre tu archivo `~/Library/LaunchAgents/ai.hermes.gateway.plist`. Si NO
   tienes Hermes corriendo como service, solo corre el comando inline (paso
   3b). Si ya lo tienes como service, paso 3a.

#### 3a — Si tienes Hermes como service (launchd)

Edita `~/Library/LaunchAgents/ai.hermes.gateway.plist` y agrega bajo
`<key>EnvironmentVariables</key>`:

```xml
<key>API_SERVER_ENABLED</key>
<string>true</string>
<key>API_SERVER_HOST</key>
<string>100.X.Y.Z</string>          <!-- ← TU TAILSCALE IP del Paso 1 -->
<key>API_SERVER_PORT</key>
<string>8642</string>
<key>API_SERVER_KEY</key>
<string>TU-KEY-DEL-PASO-2</string>  <!-- ← LA KEY del Paso 2 -->
```

Después recarga el service:

```bash
launchctl unload ~/Library/LaunchAgents/ai.hermes.gateway.plist
launchctl load ~/Library/LaunchAgents/ai.hermes.gateway.plist
```

#### 3b — Si NO tienes service (corres Hermes manual cuando quieres)

```bash
# Reemplaza 100.X.Y.Z con tu Tailscale IP del Paso 1
# Reemplaza TU-KEY-DEL-PASO-2 con la key que generaste
API_SERVER_ENABLED=true \
API_SERVER_HOST=100.X.Y.Z \
API_SERVER_PORT=8642 \
API_SERVER_KEY=TU-KEY-DEL-PASO-2 \
hermes gateway run --replace
```

Deja esa terminal ABIERTA. Si la cierras, el bridge se apaga.

### Verifica que funciona

En **OTRA terminal** del Mac:

```bash
curl http://localhost:8642/health
```

Si responde:

```json
{"status":"ok","platform":"hermes-agent"}
```

✅ El bridge está vivo. Sigue al Paso 4.

Si NO responde: copia el error y pégaselo a Claude Code. Dile:
> "Paso 3 del SETUP-FROM-ZERO falla. Mi error es: <pega>. Ayúdame."

---

## Paso 4 — Conectar tu EC2 a tu Mac

### ¿Qué hace este paso?

Le dice a tu EC2 dónde está tu Mac (Tailscale IP) y cómo identificarse
(la Bearer key del Paso 2).

### Lo que tienes que hacer

SSH a tu EC2:

```bash
ssh ec2-user@TU-EC2-HOST
```

Crea la carpeta de secrets si no existe y guarda la key:

```bash
mkdir -p ~/.hermes/secrets
printf '%s' 'TU-KEY-DEL-PASO-2' > ~/.hermes/secrets/mac-bridge-key
chmod 600 ~/.hermes/secrets/mac-bridge-key
```

Verifica desde EC2 que puedes alcanzar tu Mac:

```bash
curl --connect-timeout 5 --max-time 10 http://100.X.Y.Z:8642/health
```

Si responde `{"status":"ok","platform":"hermes-agent"}` ✅ → EC2 ve tu Mac.

Si NO responde (timeout):
- Verifica que tu Mac está DESPIERTA (no en sleep mode).
- Verifica `tailscale status` en EC2 — debería mostrar tu Mac listada.
- Verifica que el Paso 3 sigue corriendo (no cerraste la terminal).

---

## Paso 5 — Conectar Meta Agentic Bridge (MAB) si lo usas

> Si usas [Meta Agentic Bridge](https://github.com/durang/Meta-agentic-bridge),
> agrega un 4to agente "Hey Hermes Local" que rutea a tu Mac.

En tu EC2:

```bash
# Agrega las env vars a mab.env (donde mab-bridge service lee config)
echo "MAB_HERMES_LOCAL_URL=http://100.X.Y.Z:8642" | \
    sudo tee -a /home/ec2-user/meta-agentic-bridge-server/mab.env
echo "MAB_HERMES_LOCAL_KEY=TU-KEY-DEL-PASO-2" | \
    sudo tee -a /home/ec2-user/meta-agentic-bridge-server/mab.env

# Restart el bridge para que las lea
sudo systemctl restart mab-bridge

# Verifica que el 4to agente aparece
curl -s http://localhost:18790/targets | python3 -c \
    'import sys,json; print([t["id"] for t in json.load(sys.stdin)["targets"]])'
# Esperado: ['jarvis', 'hermes', 'hermes_local', 'rapido']
```

Después en MAB iOS:
1. Cmd+R en Xcode para instalar el build con el cambio
2. Abre MAB → verás 4 botones: 🦞 🪽 💻 ⚡
3. Di: **"Hey Hermes Local, qué archivos hay en mi Desktop"**
4. Tu Mac responde via TTS en lentes

---

## Si todo falla

Copia este archivo COMPLETO y pégaselo a Claude Code. Dile:
> "Sigo el SETUP-FROM-ZERO de hermes-desktop-bridge. Estoy en el Paso X y
> me trabé con esto: <descripción>. Tengo este output: <pega>. Ayúdame a
> resolverlo paso a paso."

Claude Code te va a entender el contexto completo y resolverlo contigo.

## Troubleshooting rápido

| Síntoma | Probable causa | Fix |
|---|---|---|
| `command not found: tailscale` | Tailscale no instalado | https://tailscale.com/download/mac |
| `command not found: hermes` | Hermes no instalado | Sigue el setup de Hermes primero |
| `/health` da timeout desde EC2 | Mac dormida O API_SERVER_HOST=127.0.0.1 | Verifica Paso 3 — debe ser tu Tailscale IP, no localhost |
| `401 Unauthorized` | Bearer key no coincide | Re-copia la key exacto del Paso 2 a mab.env |
| Funciona desde localhost pero NO desde otra máquina | Bind a 127.0.0.1 | Bind a 0.0.0.0 o tu Tailscale IP, no localhost |
| Mac dormida → bridge no responde | Esperado | Configura "Wake on Tailscale" en System Settings → Energy |

---

**Si llegaste aquí: tu Mac está conectada al EC2 vía Tailscale, lista para
recibir requests del cloud Hermes. Felicidades.**
