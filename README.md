# PixelFind — Google Images Application with API

Buscador de imágenes de **Google en tiempo real** con una interfaz moderna, accesible y
responsive. El frontend (React + TypeScript + Tailwind CSS) consume un backend ligero de
**Express** que actúa como proxy seguro de **SerpApi** (Google Images Results), manteniendo la
API key fuera del navegador.

> Refactorización integral del proyecto original: arquitectura en monorepo, TypeScript de punta a
> punta, estados de carga/error/vacío, lightbox accesible, paginación real, filtros, tema
> claro/oscuro y búsquedas recientes. El núcleo funcional (SerpApi) se conserva intacto.

---

## Características

- **Búsqueda en tiempo real** de Google Images vía SerpApi, con resultados JSON estructurados.
- **Galería tipo masonry** responsive con carga diferida (`lazy`) y skeletons mientras carga.
- **Lightbox accesible**: imagen original, navegación con teclado (← / → / Esc), copiar enlace,
  abrir página de origen, focus-trap y bloqueo de scroll.
- **Paginación real** usando el parámetro `ijn` de Google Images, con "cargar más" e
  _infinite scroll_ (IntersectionObserver).
- **Filtros**: tamaño, tipo, color y SafeSearch, expuestos desde el backend de SerpApi.
- **Tema claro/oscuro** persistente (sin parpadeo) y **búsquedas recientes** en `localStorage`.
- **Accesibilidad y UX**: estados explícitos (inicial, cargando, error con reintento, sin
  resultados), `alt` descriptivos, foco visible y diseño _mobile-first_.
- **Backend endurecido**: validación con Zod, `helmet`, CORS restringido, rate limiting y
  manejo de errores centralizado (sin filtrar detalles al cliente).

---

## Stack

| Capa     | Tecnologías                                                       |
| -------- | ----------------------------------------------------------------- |
| Frontend | React 19, TypeScript, Vite, Tailwind CSS v4, lucide-react         |
| Backend  | Node.js, Express 5, TypeScript (ejecutado con `tsx`), Zod, Helmet |
| API      | SerpApi — Google Images Results                                   |
| Calidad  | ESLint, Prettier, Vitest + Testing Library                        |

---

## Arquitectura

```
Navegador ──(/api/images?q=…)──▶  Express (proxy)  ──(getJson)──▶  SerpApi ──▶  Google Images
   ▲                                   │
   └──────── JSON normalizado ─────────┘   (la API key vive solo en el servidor)
```

Monorepo con **npm workspaces**:

```
google-images-app/
├─ package.json            # orquestador de workspaces (scripts dev/build/test)
├─ client/                 # SPA React + Vite + Tailwind
│  └─ src/
│     ├─ components/        # ui/ · layout/ · search/ · gallery/ · lightbox/ · states/
│     ├─ hooks/             # useImageSearch · useTheme · useRecentSearches · useLockBodyScroll
│     ├─ lib/               # api (fetch tipado) · cn
│     └─ types/             # contrato compartido con el backend
└─ server/                 # API Express + TypeScript
   └─ src/
      ├─ services/serpapi.ts   # núcleo: getJson() + normalización del DTO
      ├─ routes/images.ts      # GET /api/images, GET /api/health
      ├─ middleware/           # validate (Zod) · errorHandler
      └─ config.ts             # carga y valida variables de entorno
```

---

## Puesta en marcha

### Requisitos

- Node.js ≥ 20
- Una API key de [SerpApi](https://serpapi.com/manage-api-key)

### 1. Instalar dependencias (una sola vez, desde la raíz)

```bash
npm install
```

### 2. Configurar variables de entorno

```bash
cp server/.env.example server/.env
# edita server/.env y coloca tu SERP_API_KEY
```

### 3. Arrancar en desarrollo (cliente + servidor a la vez)

```bash
npm run dev
```

- Cliente (Vite): http://localhost:5173
- API (Express): http://localhost:3000/api

En desarrollo, Vite hace _proxy_ de `/api` hacia `localhost:3000`, así que el navegador trabaja
mismo-origen (sin CORS) y no hace falta configurar URLs.

---

## Scripts

Desde la raíz del repositorio:

| Comando             | Descripción                                       |
| ------------------- | ------------------------------------------------- |
| `npm run dev`       | Levanta cliente y servidor en paralelo            |
| `npm run build`     | Compila servidor (`tsc`) y cliente (`vite build`) |
| `npm start`         | Sirve el servidor ya compilado (`node dist`)      |
| `npm run lint`      | ESLint sobre el cliente                           |
| `npm run typecheck` | Comprobación de tipos (cliente + servidor)        |
| `npm test`          | Tests del cliente (Vitest)                        |
| `npm run format`    | Formatea el repo con Prettier                     |

---

## API

### `GET /api/images`

| Parámetro     | Tipo   | Default  | Descripción                                        |
| ------------- | ------ | -------- | -------------------------------------------------- |
| `q`           | string | —        | **Requerido.** Términos de búsqueda.               |
| `page`        | number | `0`      | Página de Google Images (`ijn`).                   |
| `imgsz`       | enum   | —        | Tamaño: `l` (grande), `m` (mediano), `i` (icono).  |
| `image_type`  | enum   | —        | `photo`, `clipart`, `lineart`, `gif`.              |
| `image_color` | enum   | —        | `red`, `blue`, `black_and_white`, `transparent`, … |
| `safe`        | enum   | `active` | SafeSearch: `active` u `off`.                      |

**Respuesta**

```jsonc
{
  "query": "gatos",
  "page": 0,
  "count": 100,
  "hasMore": true,
  "results": [
    {
      "position": 1,
      "title": "Felis catus - Wikipedia",
      "thumbnail": "https://…",
      "original": "https://…",
      "source": "Wikipedia",
      "link": "https://…",
      "width": 1824,
      "height": 1812,
    },
  ],
}
```

### `GET /api/health`

Comprobación de readiness: `{ "status": "ok", "timestamp": "…" }`.

---

## Variables de entorno (servidor)

| Variable                | Default                 | Descripción                              |
| ----------------------- | ----------------------- | ---------------------------------------- |
| `SERP_API_KEY`          | —                       | **Requerida.** Clave de SerpApi.         |
| `PORT`                  | `3000`                  | Puerto del servidor.                     |
| `CLIENT_ORIGIN`         | `http://localhost:5173` | Orígenes permitidos por CORS (coma-sep). |
| `SERPAPI_LOCATION`      | `Mexico`                | Localización por defecto de la búsqueda. |
| `SERPAPI_GOOGLE_DOMAIN` | `google.com.mx`         | Dominio de Google.                       |
| `SERPAPI_HL` / `_GL`    | `es` / `mx`             | Idioma y país.                           |

El cliente admite `VITE_API_BASE_URL` para apuntar a la API en producción (en dev se usa el proxy).

> ⚠️ **Seguridad:** `server/.env` está ignorado por Git. Si alguna vez una clave quedó commiteada,
> **rótala** desde el panel de SerpApi.

---

## Calidad

```bash
npm run lint        # ESLint
npm run typecheck   # TypeScript (cliente + servidor)
npm test            # Vitest (api, reducer de búsqueda, componentes)
```

---

## Créditos

Datos de imágenes proporcionados por [SerpApi — Google Images Results](https://serpapi.com).
