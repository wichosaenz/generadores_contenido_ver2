# Guía de Implementación — WF2 Everest Worker v4.0 FINAL

## Análisis del Problema de Autenticación

### Causa Raíz

Los 3 nodos Code que hacen HTTP requests a WordPress usaban este patrón:

```javascript
await this.helpers.httpRequest({
  // ...
  auth: {
    username: creds.wp_user,
    password: creds.wp_app_password
  }
});
```

**El problema:** `this.helpers.httpRequest()` en n8n es un wrapper sobre la librería `got`. El objeto `auth: { username, password }` **no es traducido automáticamente** a un header `Authorization: Basic`. El request llega a WordPress **sin autenticación** → respuesta `401 Unauthorized`.

### La Solución Aplicada

Se reemplazó el objeto `auth` con un header `Authorization` construido manualmente usando Base64:

```javascript
const authHeader = 'Basic ' + Buffer.from(
  creds.wp_user + ':' + creds.wp_app_password
).toString('base64');

await this.helpers.httpRequest({
  // ...
  headers: {
    'Authorization': authHeader,
    'Content-Type': 'application/json'
  }
  // SIN objeto auth
});
```

Esta solución es **agnóstica a la versión de n8n** porque controla directamente el header HTTP sin depender del comportamiento interno del wrapper.

### Correcciones Adicionales

| # | Nodo | Fix |
|---|------|-----|
| 1 | Code - Upload Media to WP | `auth` → `Authorization` header manual |
| 2 | Code - Create WP Post | `auth` → `Authorization` header manual |
| 3 | Code - PATCH RankMath SEO | `auth` → `Authorization` header manual |
| 4 | Agent - Escritor | Agregado `systemMessage: "={{ $json.system_prompt_final }}"` |
| 5 | Agent - Director de Arte | Agregado `systemMessage: "={{ $json.art_director_system_prompt }}"` |

---

## Pasos de Importación

### 1. Importar el Workflow

1. Abrir n8n → Menu lateral → **Workflows**
2. Click **Import from File**
3. Seleccionar `WF2_Worker_FINAL.json`
4. El workflow aparecerá como **"WF2 - Everest Worker v4.0 FINAL"**

### 2. Asignar Credenciales

Después de importar, n8n mostrará advertencias en los nodos que requieren credenciales. Debes registrar y asignar las siguientes:

| Credencial en n8n | Tipo | Nodos que la usan | Notas |
|---|---|---|---|
| Anthropic API | `anthropicApi` | Claude - Query Generator, Claude - Agent - Core Data, Claude - Agent - Hermanos, Claude - Agent - Claims Antitesis, Claude - Escritor | API key de Anthropic |
| OpenAI API | `openAiApi` | Embeddings (x3), OpenAI - Director de Arte | API key de OpenAI |
| Pinecone API | `pineconeApi` | Pinecone - Agent - Core Data, Hermanos, Claims Antitesis | API key de Pinecone |
| Google Gemini (PaLM) | `googlePalmApi` | Generate an image | API key de Google AI Studio |

**Las credenciales de WordPress NO se registran en n8n.** Llegan dinámicamente en el payload desde WF1 Dispatcher vía Google Sheets.

### 3. Vincular con WF1 Dispatcher

En WF1, el nodo **"Execute Workflow - Worker"** necesita el ID del WF2:
1. Abrir WF2 en n8n
2. Copiar el ID del workflow (visible en la URL: `https://tu-n8n.com/workflow/XXXXX`)
3. En WF1, editar el nodo "Execute Workflow - Worker" → campo `workflowId` → pegar el ID

---

## Verificar que la Auth Dinámica Funciona

### Test Rápido (sin ejecutar todo el pipeline)

1. Abrir **WF2** en n8n
2. El nodo **Execute Workflow Trigger** ya tiene `pinData` (datos de prueba) con credenciales reales del sitio `S2_MexicoFulfillmentExperts`
3. Crear un nodo **Code** temporal conectado después de "Generate an image" con este código:

```javascript
// Test de autenticación dinámica
const creds = $('Execute Workflow Trigger').first().json;
const wpUrl = creds.wp_url.replace(/\/$/, '');
const authHeader = 'Basic ' + Buffer.from(
  creds.wp_user + ':' + creds.wp_app_password
).toString('base64');

try {
  const response = await this.helpers.httpRequest({
    method: 'GET',
    url: `${wpUrl}/wp-json/wp/v2/users/me`,
    headers: {
      'Authorization': authHeader
    },
    returnFullResponse: false
  });
  return [{ json: { auth_ok: true, user: response.name, id: response.id } }];
} catch (error) {
  return [{ json: { auth_ok: false, error: error.message } }];
}
```

4. Ejecutar solo ese nodo. Si devuelve `auth_ok: true`, la autenticación funciona.

### Test Completo

1. Ejecutar WF2 completo con los datos pineados
2. Verificar en `wp-admin` del sitio:
   - **Media Library:** imagen subida con nombre `S2_MexicoFulfillmentExperts-{slug}-{timestamp}.png`
   - **Posts:** post programado con status "Scheduled"
   - **RankMath SEO:** meta title, description y focus keyword configurados

---

## Requisitos de WordPress

Para que la auth dinámica funcione, cada sitio WordPress debe tener:

1. **Application Passwords habilitados** (WordPress 5.6+, activado por defecto)
2. **Usuario API** con permisos de `editor` o `administrator`
3. **Application Password generada** para ese usuario:
   - wp-admin → Users → Edit User → Application Passwords
   - Generar y guardar en la Google Sheet `Sitios_Config`
4. **REST API accesible** (no bloqueada por plugin de seguridad)
5. **RankMath SEO instalado** (para el PATCH de meta SEO)

---

## Troubleshooting

| Error | Causa | Solución |
|-------|-------|----------|
| `401 Unauthorized` en Upload/Post/PATCH | Application Password incorrecta o usuario sin permisos | Verificar `wp_user` y `wp_app_password` en Google Sheet. Regenerar Application Password si es necesario |
| `403 Forbidden` | El usuario no tiene permisos de publicación, o un plugin de seguridad bloquea la REST API | Verificar que el usuario tenga rol `editor` o superior. Revisar plugins como Wordfence, iThemes Security |
| `rest_cannot_create` | El usuario no tiene capability `publish_posts` | Cambiar rol del usuario a `editor` o `administrator` |
| `404 Not Found` en `/wp-json/wp/v2/...` | Permalinks no configurados o REST API deshabilitada | Ir a Settings → Permalinks → guardar sin cambiar. Verificar que ningún plugin deshabilite la REST API |
| `400 Bad Request` en Create Post | Campo `date` con formato incorrecto para status `future` | Verificar que `wp_scheduled_date` tenga formato `YYYY-MM-DDTHH:MM:SS` y sea una fecha futura |
| `No binary data from Gemini` | El nodo Gemini no devolvió imagen en formato binario | Verificar credenciales de Google AI. Comprobar que el modelo `imagen-4.0-ultra-generate-001` esté disponible |
| `Director de Arte no generó un prompt válido` | GPT-4o-mini devolvió texto no parseable como JSON | El nodo Code tiene fallback que usa el texto raw. Si persiste, verificar que el system prompt del Art Director se inyecte correctamente |
| `RankMath PATCH failed` con `rest_no_route` | RankMath no está instalado o la REST API de RankMath está deshabilitada | Instalar/activar RankMath SEO en el sitio WordPress |
| System prompt del Escritor vacío | `system_prompt_final` no llega en el payload | Verificar en WF1 que la hoja `Perfiles_Escritores` tiene el campo y que el Code de Cross JOIN lo incluye |

---

## Arquitectura de Referencia

```
Execute Workflow Trigger (payload de WF1)
    │
    ▼
Set - Prepare System Prompt
    │
    ▼
Agent - Query Generator (Claude Sonnet 4)
    │
    ▼
Code - Prepare RAG Input
    │
    ├──▶ Agent - Core Data ────────┐
    ├──▶ Agent - Hermanos ─────────┤ (paralelo)
    └──▶ Agent - Claims Antitesis ─┘
                                   │
                                   ▼
                          Merge - RAG Results
                                   │
                                   ▼
                     Code - Build Writer Input
                                   │
                                   ▼
                     Agent - Escritor (Claude Sonnet 4, 16K tokens)
                     + Structured Output Parser
                                   │
                                   ▼
                     Set - Extract Output & Creds
                                   │
                                   ▼
                     Agent - Director de Arte (GPT-4o-mini)
                                   │
                                   ▼
                     Code - Extract Image Prompt
                                   │
                                   ▼
                     Generate an image (Gemini Imagen 4.0 Ultra)
                                   │
                                   ▼
                     Code - Upload Media to WP ──── POST /wp/v2/media
                                   │
                                   ▼
                     Code - Create WP Post ──────── POST /wp/v2/posts
                                   │
                                   ▼
                     Code - PATCH RankMath SEO ──── PATCH /wp/v2/posts/{id}
```
