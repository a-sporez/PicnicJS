# PicnicJS

**DOTENV Variables need to be created to authorize**

Currently the only functionality is a bridge between a discord bot and VPS or localhost. It should work for any LLM with minor adjustments but it has only been tested on Digital Ocean Mistral Nemo Instruct model. Current interface are express and openAPI.

Modern JS plugin-based web app.

## Component Map

                       +-------------------------+
                       |      Frontend App       |
                       |         (Vite)          |
                       +-----------+-------------+
                                   |
                        HTTPS/API Calls / WebSocket
                                   ↓
           +----------------------↓----------------------+
           |                  API Gateway                |
           |          express-rate-limit + proxy         |
           +----------------------↓----------------------+
                        Auth Headers (JWT from Keycloak)
                                   ↓
                       +----------↓-----------+
                       |   Module Host (API)  |
                       |   main.ts            |
                       +--+-----+------+------+
                          |     |      |
                          ↓     ↓      ↓
               +----------+ +----------+ +-------------+
               | StreamCfg | | ClientCfg| | ChatModule |
               | Service   | | Service  | | EmoteMod   |
               +-----+-----+ +-----+----+ +------+------+
                     |             |             |
                     ↓             ↓             ↓
               Redis Cloud     PostgreSQL     In-memory/Redis

                   ⬑ Shared Redis Pub/Sub Queue (event bus)

                          +-------------------+
                          |  Keycloak Server  |
                          | (OAuth2 + RBAC)   |
                          +-------------------+

## Technologies and Tools to 

| Layer             | Stack / Tools                                    |
| ----------------- | ------------------------------------------------ |
| **Frontend**      | Go, Vite CSS, SWR (data), WebSockets             |
| **Auth**          | Keycloak (OIDC/JWT), OAuth2.1 compatible flows   |
| **Backend**       | Node.js (npm), Express API                       |
| **Plugins**       | `Command`, `Emote`, `StreamConfigService`, `CLI` |
| **Database**      | Redis (stream/user state), RAG knowledge base    |
| **Communication** | WebSocket or Redis Pub/Sub                       |
| **Rate Limits**   | `express-rate-limit`, or Cloudflare WAF          |
| **Storage**       | Optional: S3/Cloudflare R2 for plugin uploads    |

**DOCUMENTATION IS INCOMPLETE**

---

## Core Systems

### `core/context.js`

| Name                  | Type       | Description                                           |
| --------------------- | ---------- | ----------------------------------------------------- |
| `create_context()`    | `Function` | Creates a shared context object passed to all plugins |
| `context.emit(event)` | `Function` | Emits an event to the internal event bus              |
| `context.logger`      | `Object`   | Logger (uses `console` for now)                       |
| `context.hostId`      | `String`   | Identifier for this host instance                     |

---

### `core/event_bus.js`

| Name             | Type              | Description                                    |
| ---------------- | ----------------- | ---------------------------------------------- |
| `subscribe(fn)`  | `Function`        | Adds a subscriber (callback) to receive events |
| `publish(event)` | `Function`        | Broadcasts an event to all subscribers         |
| `subscribers`    | `Array<Function>` | Internal list of event listeners               |

---

### `core/plugin_manager.js`

| Name                    | Type       | Description                                                       |
| ----------------------- | ---------- | ----------------------------------------------------------------- |
| `load_plugins(context)` | `Function` | Loads all plugins from `plugins/` directory, calls their `init()` |

---

## Entrypoint

### `main.js`

| Name                | Type            | Description                                            |
| ------------------- | --------------- | ------------------------------------------------------ |
| `main()`            | `Function`      | Loads context, loads plugins, subscribes to all events |
| `context.emit(...)` | `Function`      | Emits events manually (simulated test cases)           |
| `plugins`           | `Array<Plugin>` | List of loaded plugin modules                          |

---

## Express API

### `server.js`

| Name                    | Type    | Description                                                     |
| ----------------------- | ------- | --------------------------------------------------------------- |
| `POST /events/incoming` | `Route` | Accepts an event `{ type, payload }` and injects into event bus |

---

## Plugins

### `chat_module/index.js`

| Name                                               | Type       | Description                                           |
| -------------------------------------------------- | ---------- | ----------------------------------------------------- |
| `chatLog`                                          | `Array`    | Stores all received chat messages                     |
| `context.emit({ type: 'chat::message:received' })` | `Function` | Broadcasts a processed message                        |
| `handle_event()`                                   | `Function` | Receives `chat::message`, pushes to log, rebroadcasts |

---

### `emote_module/index.js`

| Name                                             | Type       | Description                                                        |
| ------------------------------------------------ | ---------- | ------------------------------------------------------------------ |
| `EMOTE_MAPS`                                     | `Object`   | Dictionary mapping emote text to emoji                             |
| `handle_event()`                                 | `Function` | Receives `chat::message:received`, transforms emotes, rebroadcasts |
| `context.emit({ type: 'chat::message:emoted' })` | `Function` | Broadcasts transformed message                                     |

---

### `stream_config/index.js`

| Name                      | Type       | Description                                           |
| ------------------------- | ---------- | ----------------------------------------------------- |
| `streamConfig`            | `Object`   | Holds stream metadata (`title`, `tags`)               |
| `handle_event()`          | `Function` | Handles `stream::config:update`, `stream::config:get` |
| `context.logger.log(...)` | `Function` | Outputs stream config status                          |

---

### `client_config/index.js`

| Name                         | Type       | Description                                 |
| ---------------------------- | ---------- | ------------------------------------------- |
| `configs`                    | `Map`      | In-memory map of `clientId` → config object |
| `contextReference`           | `Object`   | Stores context for use in `handle_event`    |
| `handle_event()`             | `Function` | Handles client config updates and lookups   |
| `contextReference.emit(...)` | `Function` | Broadcasts `client::config:changed`         |

---

### `greeter/index.js`

| Name             | Type       | Description                                   |
| ---------------- | ---------- | --------------------------------------------- |
| `handle_event()` | `Function` | Responds to `hello` events for debug purposes |

---

### Example: Event Object

```js
{
  type: 'chat::message',
  payload: {
    clientId: 'abc123',
    user: 'alice',
    message: ':fire: hello'
  },
  meta: {
    plugin: 'chat_module'
  }
}
```

---
TODO:

- Mistral Bridge Docs
- Discord Bridge Docs
- Server Module Docs
