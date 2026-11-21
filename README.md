# KI:Link Portal — Customised Open WebUI (SvelteKit)

A white-labelled, simplified chat portal built on **Open WebUI v0.11.3**. The SvelteKit frontend was modified to present a minimal, focused chat experience for the **KI Connect NRW** LLM backend, while the Python backend remains unchanged.

| Item | Value |
|---|---|
| Base project | [open-webui/open-webui](https://github.com/open-webui/open-webui) `v0.11.3` |
| Custom branch | `dev` (4 commits on top of `v0.11.3`) |
| Frontend | SvelteKit + Tailwind CSS |
| Backend LLM provider | KI Connect NRW (OpenAI-compatible API) |
| Docker image | `ki-link-portal:v1.0-slim` |

## Repository layout

```
custom_dev/
├── docker-compose.yaml        # Runtime deployment (see below)
├── open-webui-data/           # Persistent data volume (created at runtime)
└── custom-ki-portal/          # Modified Open WebUI source tree
    ├── src/                   # SvelteKit frontend (modified)
    ├── static/logo_ki_link.png# KI:Link logo (new)
    ├── backend/               # Open WebUI backend (unmodified)
    └── Dockerfile
```

## Features and modifications

### 1. Simplified navigation and chat UI — `782e69d`
- Sidebar reduced to **New Chat**, **Search chats** and **Chat history**. Notes, Workspace, Channels, Prompts, Documents and Tools entries were removed.
- Footer branding and non-essential external links hidden.
- Voice input, dictation and call mode removed from the message input.
- Suggested prompts below the input removed.

Files: `Sidebar.svelte`, `Chat.svelte`, `ChatPlaceholder.svelte`, `Placeholder.svelte`, `MessageInput.svelte`

### 2. KI:Link branding — `3ff229b`, `49840fc`
- New logo (`static/logo_ki_link.png`) used for the favicon, browser tab, splash screen, sidebar, sign-in page, assistant/user avatars and chat placeholder (new `Logo.svelte` component).
- Application name fixed to **KI:Link** (page title, `WEBUI_NAME` store, sign-in/sign-up/LDAP headings, navbar title).
- Model selector hides embedding models (`embed`, `e5-`, `bge-`) so only chat models can be selected.

Files: `app.html`, `+layout.svelte`, `auth/+page.svelte`, `Logo.svelte`, `Navbar.svelte`, `ModelSelector.svelte`, `ProfileImage.svelte`, `ResponseMessage.svelte`, `UserMessage.svelte`

### 3. Hidden reasoning and tool-call output — `3ff229b`
- Reasoning blocks and tool-call dropdowns are no longer rendered in the chat. Only tool calls awaiting user approval (`pending`) and `ask_user` remain visible.
- Web search progress is shown as a simple **"Searching the web…"** spinner instead of the expandable step-by-step status history.

Files: `MarkdownTokens.svelte`, `structuredOutput.ts`, `StatusHistory.svelte`, `StatusItem.svelte`, `InputMenu.svelte`

### 4. Optional API key at sign-in — `49840fc`, `c84e88c`
- The login page has an optional, masked **API Key** field.
- **Admin users:** the key is written to the backend OpenAI connection (Admin Panel → Settings → Connections) via `getOpenAIConfig` / `updateOpenAIConfig`.
- **Other users:** the key is stored in their personal *direct connections* settings via `getUserSettings` / `updateUserSettings`.
- Fixed the admin-panel API connection handling and the Open WebUI splash flashing on sign-out (custom logo now used in the splash).

Files: `routes/auth/+page.svelte`, `app.html`, `StatusHistory.svelte`

## Prerequisites

- Linux host with Git and Docker (with BuildKit)
- Node.js 20 (project supports `>=18.13 <=22`)
- A KI Connect API key

## Development setup

```bash
# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

cd custom-ki-portal
npm install
npm run dev        # Vite dev server with hot reload
npm run build      # Production frontend build
```

## Docker image build

A slim, CPU-only image without bundled Ollama is built. `BUILD_MAX_CONCURRENCY=1` limits parallelism to reduce memory use.

```bash
cd custom-ki-portal
BUILD_MAX_CONCURRENCY=1 docker build \
  --build-arg USE_SLIM=true \
  --build-arg USE_CUDA=false \
  --build-arg USE_OLLAMA=false \
  -t ki-link-portal:v1.0-slim .
```

## Deployment

Create `docker-compose.yaml` next to the `custom-ki-portal` folder (replace the key placeholders with your own and never commit real keys):

```yaml
services:
  open-webui:
    image: ki-link-portal:v1.0-slim
    container_name: ki-link-portal
    ports:
      - "8080:8080"
    volumes:
      - ./open-webui-data:/app/backend/data
    environment:

      # Prevent HuggingFace downloads
      - HF_HUB_OFFLINE=1

      # Disable unused features
      - ENABLE_OLLAMA_API=False
      - ENABLE_IMAGE_GENERATION=False
      - ENABLE_COMMUNITY_SHARING=False

      # KI Connect Backend Connection
      - OPENAI_API_BASE_URL=https://chat.kiconnect.nrw/api/v1
      - OPENAI_API_KEY=<YOUR_KICONNECT_VAPI_KEY>

      # RAG Embeddings via KI Connect
      - RAG_EMBEDDING_ENGINE=openai
      - RAG_OPENAI_API_BASE_URL=https://chat.kiconnect.nrw/api/v1
      - RAG_OPENAI_API_KEY=<YOUR_KICONNECT_VAPI_KEY>
      - RAG_EMBEDDING_MODEL=qwen3-embedding-8b

      # Web Search (DuckDuckGo)
      - ENABLE_WEB_SEARCH=True
      - WEB_SEARCH_ENGINE=DDGS

      # Portal Identity & Auth
      - WEBUI_NAME=KI:Link
      - WEBUI_AUTH=True
      - DEFAULT_USER_ROLE=user
    restart: unless-stopped
```

Start and access:

```bash
docker compose up -d
docker compose logs -f open-webui
```

Open <http://localhost:8080>. The first account registered becomes the administrator.

### Configuration summary

| Variable | Purpose |
|---|---|
| `OPENAI_API_BASE_URL` / `OPENAI_API_KEY` | Chat completions via KI Connect |
| `RAG_EMBEDDING_ENGINE=openai`, `RAG_EMBEDDING_MODEL` | Document embeddings via KI Connect (`qwen3-embedding-8b`), no local model |
| `HF_HUB_OFFLINE=1` | No HuggingFace downloads (offline-safe) |
| `ENABLE_OLLAMA_API`, `ENABLE_IMAGE_GENERATION`, `ENABLE_COMMUNITY_SHARING` | Disabled unused features |
| `ENABLE_WEB_SEARCH`, `WEB_SEARCH_ENGINE=DDGS` | DuckDuckGo web search |
| `WEBUI_AUTH`, `DEFAULT_USER_ROLE` | Authentication required; new users get the `user` role |

## Commit history

| Commit | Description |
|---|---|
| `782e69d` | Simplified sidebar, chat placeholder and message input |
| `3ff229b` | Added logos; hid reasoning UI and search steps |
| `49840fc` | Auth API key input and logo fixes |
| `c84e88c` | Admin-panel API connection fix; sign-out splash fix |

## License note

This project is a derivative of Open WebUI and is subject to the [Open WebUI License](custom-ki-portal/LICENSE). Clause 4 prohibits altering or removing Open WebUI branding unless the deployment has **no more than 50 end users in any rolling 30-day period**, or written permission / an enterprise license has been obtained. The rebranding described here is intended for academic, limited-user use; obtain permission or a license before any wider deployment.
