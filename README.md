# chatcorporativo.djlabs.cl — front independiente (Chat Corporativo)

Front público del producto **Chat Corporativo**: un asistente de IA que
responde en la **página web** del cliente y en su **WhatsApp**, usando
únicamente la información que el cliente subió a su espacio.

```
Navegador ──REST JSON + cookies──▶ plataforma (corazón chatcorporativo)
   │
   └── CORS: el backend debe permitir este origen exacto
```

Nada de esto se sirve desde la plataforma: el front tiene su propio
document root, sus propias rutas (todas relativas) y su propia perilla de
despliegue.

## Qué hay aquí

| Archivo | Qué hace |
|---|---|
| `index.html` | La landing: hero, captura de contenido, cómo funciona, qué ofrece, acceso (login/registro), recargas, contacto y los modales de términos/privacidad. |
| `app.html` | El panel: información del cliente, canales, asistente, billetera, recargas, fuentes del RAG, conversaciones de visitantes, perfil y uso. |
| `contrasena.html`, `verificar.html` | Cerrar sesión con contraseña y confirmar el correo. |
| `debug-sources.html` | Diagnóstico de las fuentes del RAG (para soporte). |
| `js/` | Los módulos compartidos con la plataforma (`chat-widget`, `checkout`, `topup`, `notifications`, `uso`, …), hablando por `apiUrl()`. |
| `css/tokens.css`, `css/quill.snow.css` | El sistema de diseño (la piel la define el heart, por REST). |
| `api-config.js` | **La única perilla del despliegue**: el origen del backend para el navegador. |

## El recorrido del cliente

1. **Crea su cuenta** en la landing (100 créditos de bienvenida, sin tarjeta).
2. **Sube su información**: archivos (catálogo, precios, preguntas frecuentes,
   políticas) y/o captura de contenido desde una URL. Esa información se
   guarda como **base de conocimiento** de su espacio, en su propio corazón.
3. **Activa sus canales**: la tarjeta *Tus canales* entrega el código de
   inserción del chat web y la sección de WhatsApp (con su número y QR).
4. **Prueba el asistente** en el panel antes de publicarlo.
5. **Atiende**: las conversaciones de los visitantes de ambos canales quedan
   visibles en el panel.

## Configuración

`api-config.js` (un solo archivo, se reescribe en el deploy):

```js
window.__API_BASE__ = 'https://api.djlabs.cl';
```

El valor es **fijo**: el front siempre habla con la API, nunca consigo mismo.
Dos cosas para que funcione:

1. El origen del front debe estar en la allowlist `cors.origins` del backend
   (coincidencia exacta de esquema+host+puerto).
2. La cookie de sesión debe viajar: `api.djlabs.cl` y `*.djlabs.cl` son el
   **mismo sitio** (mismo dominio registrable), así que `SameSite=Lax` +
   HTTPS alcanza — no hace falta `None`.

**En local** el origen **no** se cambia desde la URL: el override `?api=...`
se eliminó (2026-11-09) porque podía dejar un front ya desplegado apuntando a
un origen muerto — fue exactamente el bug que dejó el chat sin conexión en
producción, cuando un `api-config.js` cacheado apuntaba a
`http://127.0.0.1:8099`. Para correr contra una API local, el arnés E2E
inyecta `window.__API_BASE__` antes de cargar la página.

> Ojo: la allowlist real la fija la variable de entorno `API_ORIGINS` del
> backend, que **pisa** el `cors.origins` del archivo de configuración. Un
> origen que no esté en `API_ORIGINS` recibe un preflight fallido (404) y el
> navegador reporta `Failed to fetch`.

El **heart** no se configura aquí: el clon trabaja con `chatcorporativo`, el
corazón del producto. La piel (`esmeralda`) y el tema los aplica el backend
(REST `/ui/theme?heart=chatcorporativo`), igual que en la plataforma.

## Cómo probarlo

```bash
# En local, contra la plataforma real
php -S 127.0.0.1:8811 -t .

# Batería E2E en el navegador (registro → panel → adjunto → chat → canales)
node ../tests/cdp_chatcorporativo.mjs
```

La batería E2E recorre el camino completo del cliente: la landing apunta al
corazón correcto, el registro abre el panel, el archivo subido queda en el
RAG de `chatcorporativo` (y **no** en el `chat-corporativo` legacy), el
asistente responde con el precio que está en el catálogo, y el snippet de
inserción incluye el `heart` correcto. Además exige cero errores de consola
y cero respuestas 4xx/5xx.

## Las reglas que este front no rompe

- **Cero `/api/*.php`**: el panel, el chat, las fuentes, el TTS y la
  descarga de documentos van por REST JSON contra la plataforma.
- **El asistente se presenta con el nombre de SU heart**: el backend resuelve
  el nombre y la persona por el heart de la petición.
- **La pausa por verificación de correo se explica**: si la cuenta está en
  pausa, el chat muestra el aviso y el botón para reenviar el enlace (nada
  de errores genéricos).
- **Los errores se explican sin culpar a nadie** y sin dejar pantallas en
  blanco.
- **El vocabulario comercial del cliente no se anonimiza**: los nombres de
  sus productos y planes ("Plan Pro") llegan textuales al modelo —eso lo
  habilita el ajuste `commercial_catalog` del corazón— mientras que los
  nombres de **personas** y los RUT siguen enmascarados antes de salir de
  nuestros servidores.
