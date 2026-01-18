# Architecture

Technical documentation for Sure Quick Entry.

---

## System Overview

Sure Quick Entry is a lightweight Progressive Web App (PWA) that provides a mobile-optimized interface for logging transactions to Sure Finance.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                    │
│                                                                          │
│    ┌──────────────┐                      ┌───────────────────────────┐  │
│    │    Phone     │                      │      Cloudflare           │  │
│    │   Browser    │ ────── HTTPS ──────▶ │   ┌─────────────────┐     │  │
│    │    (PWA)     │                      │   │   Zero Trust    │     │  │
│    └──────────────┘                      │   │  (Auth Layer)   │     │  │
│                                          │   └────────┬────────┘     │  │
│                                          │            │              │  │
│                                          │   ┌────────▼────────┐     │  │
│                                          │   │    Tunnel       │     │  │
│                                          │   │  (Encrypted)    │     │  │
│                                          │   └────────┬────────┘     │  │
│                                          └────────────┼──────────────┘  │
└──────────────────────────────────────────────────────┼──────────────────┘
                                                       │
┌──────────────────────────────────────────────────────┼──────────────────┐
│                         HOME NETWORK (Behind CG-NAT)  │                  │
│                                                       │                  │
│    ┌──────────────────────────────────────────────────▼───────────────┐ │
│    │                    Docker Host (HP EliteDesk)                     │ │
│    │                                                                   │ │
│    │   ┌─────────────────────────────────────────────────────────┐    │ │
│    │   │                  sure-quick container                    │    │ │
│    │   │                     (nginx:alpine)                       │    │ │
│    │   │                                                          │    │ │
│    │   │    ┌──────────────┐        ┌──────────────────────┐     │    │ │
│    │   │    │  Static      │        │    Reverse Proxy     │     │    │ │
│    │   │    │  Files       │        │    /api/* → Sure     │     │    │ │
│    │   │    │  (HTML/PWA)  │        │    Finance API       │     │    │ │
│    │   │    └──────────────┘        └───────────┬──────────┘     │    │ │
│    │   │                                        │                 │    │ │
│    │   └────────────────────────────────────────┼─────────────────┘    │ │
│    │                                            │                      │ │
│    │   ┌────────────────────────────────────────▼─────────────────┐    │ │
│    │   │               sure-finance container                      │    │ │
│    │   │              (Ruby on Rails + PostgreSQL)                 │    │ │
│    │   └───────────────────────────────────────────────────────────┘    │ │
│    │                                                                   │ │
│    └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. Progressive Web App (index.html)

A single HTML file containing the entire frontend application.

**Technology Stack:**
- React 18 (loaded from unpkg CDN)
- Tailwind CSS (loaded from CDN)
- Babel (in-browser JSX transpilation)

**Why Single File?**
- Simplicity: No build step required
- Deployment: Just copy one file
- Maintenance: Everything in one place

**Trade-offs:**
- CDN dependencies (supply chain consideration)
- In-browser transpilation (minor performance cost)
- No code splitting (acceptable for small app)

### 2. Nginx Proxy

Serves static files and proxies API requests to Sure Finance.

**Configuration:**
```nginx
server {
    listen 80;
    
    # Serve PWA files
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
    
    # Proxy API requests to Sure Finance
    location /api/ {
        proxy_pass https://finance.mayavasko.uk/api/;
        proxy_set_header Host finance.mayavasko.uk;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-API-Key $http_x_api_key;
        proxy_pass_header X-API-Key;
        proxy_ssl_server_name on;
    }
}
```

**Why Nginx Proxy?**
- Solves CORS issues (same-origin API calls)
- Keeps API URL simple (`/api/` vs full URL)
- Lightweight Alpine container

### 3. Sure Finance API

The backend Rails application (separate project).

- **GitHub**: https://github.com/we-promise/sure
- **Auth**: API Key via `X-API-Key` header
- **License**: AGPL-3.0

---

## Data Flow

### Authentication Flow

```
1. User opens https://quick.example.com
          │
          ▼
2. Cloudflare Zero Trust intercepts
          │
          ├─── Not authenticated? ──▶ Redirect to login
          │
          ▼
3. Authenticated → Allow through to Nginx
          │
          ▼
4. Nginx serves index.html
          │
          ▼
5. PWA checks localStorage for API key
          │
          ├─── No key? ──▶ Show config screen
          │
          ▼
6. PWA makes API calls with X-API-Key header
```

### Transaction Flow

```
1. User enters transaction details
          │
          ▼
2. React state holds form data (in-memory)
          │
          ▼
3. User taps "Add Expense/Income"
          │
          ▼
4. PWA sends POST to /api/v1/transactions
   Headers: { "X-API-Key": "...", "Content-Type": "application/json" }
   Body: { "transaction": { ... } }
          │
          ▼
5. Nginx proxies to Sure Finance
          │
          ▼
6. Sure Finance validates API key and creates transaction
          │
          ▼
7. Success response → PWA shows confirmation
          │
          ▼
8. PWA refreshes account balances
```

---

## File Structure

### Container Volume Mounts

```
Host: /srv/docker/sure-quick/          Container: /usr/share/nginx/html/
├── index.html         ─────────────▶  ├── index.html
├── manifest.json      ─────────────▶  ├── manifest.json
├── icon-192.png       ─────────────▶  └── icon-192.png
└── nginx.conf         ─────────────▶  /etc/nginx/conf.d/default.conf
```

### PWA Manifest

```json
{
  "name": "Sure Quick Entry",
  "short_name": "Quick Entry",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#0a0a0a",
  "theme_color": "#1c1c1e",
  "icons": [
    {
      "src": "icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    }
  ]
}
```

---

## Docker Configuration

### docker-compose.yml

```yaml
services:
  sure-quick:
    image: nginx:alpine
    container_name: sure-quick
    restart: unless-stopped
    volumes:
      - ./index.html:/usr/share/nginx/html/index.html:ro
      - ./manifest.json:/usr/share/nginx/html/manifest.json:ro
      - ./icon-192.png:/usr/share/nginx/html/icon-192.png:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - stack_default

networks:
  stack_default:
    external: true
```

**Notes:**
- Read-only mounts (`:ro`) for security
- Uses external network to communicate with other containers
- Lightweight Alpine-based image

---

## API Integration

### Endpoints Used

| Endpoint | Method | Request | Response |
|----------|--------|---------|----------|
| `/api/v1/accounts` | GET | - | `{ accounts: [...] }` |
| `/api/v1/categories` | GET | - | `{ categories: [...] }` |
| `/api/v1/merchants` | GET | - | `[...]` |
| `/api/v1/tags` | GET | - | `[...]` |
| `/api/v1/transactions` | POST | Transaction object | Created transaction |

### Authentication

All requests include:
```
X-API-Key: <api_key_from_localStorage>
```

### Transaction Object

```json
{
  "transaction": {
    "account_id": "550e8400-e29b-41d4-a716-446655440000",
    "date": "2026-01-18",
    "amount": 25.50,
    "name": "Coffee at Starbucks",
    "nature": "expense",
    "category_id": "550e8400-e29b-41d4-a716-446655440001",
    "merchant_id": "550e8400-e29b-41d4-a716-446655440002",
    "tag_ids": ["550e8400-e29b-41d4-a716-446655440003"],
    "notes": "Morning coffee"
  }
}
```

---

## Network Topology

### Cloudflare Configuration

**Tunnel Route:**
```yaml
- hostname: quick.example.com
  service: http://sure-quick:80
```

**Zero Trust Application:**
- Self-hosted application
- Policy: Require authentication (email/SSO)
- Session duration: Configurable

### Internal Network

The container communicates via Docker network:
- Network: `stack_default` (external)
- Container name resolution: `sure-quick`
- Internal port: 80

---

## State Management

### localStorage

```javascript
// Stored data
{
  "sureConfig": {
    "url": "",        // Not currently used (hardcoded proxy)
    "token": "..."    // API Key
  }
}
```

### React State

```javascript
// Runtime state (not persisted)
{
  config: { url, token },
  showConfig: boolean,
  isConfigured: boolean,
  accounts: [],
  categories: [],
  merchants: [],
  tags: [],
  loading: boolean,
  submitting: boolean,
  message: { type, text } | null,
  showMore: boolean,
  transaction: {
    nature: 'expense' | 'income',
    amount: string,
    name: string,
    account_id: string,
    category_id: string,
    merchant_id: string,
    tag_ids: [],
    date: string,
    notes: string
  }
}
```

---

## Performance Considerations

| Aspect | Impact | Notes |
|--------|--------|-------|
| CDN scripts | ~200ms load | Cached after first load |
| Babel transpilation | ~50ms | One-time on page load |
| API calls | Variable | Depends on network/server |
| Nginx proxy | Minimal | Single hop on same host |

### Optimization Opportunities

1. Bundle dependencies locally (trade-off: larger file, no CDN caching)
2. Pre-transpile JSX (trade-off: requires build step)
3. Add service worker for offline support (future enhancement)

---

## Deployment Checklist

- [ ] Copy files to deployment directory
- [ ] Update nginx.conf with correct Sure Finance URL
- [ ] Create app icon (192x192 PNG)
- [ ] Configure Cloudflare Tunnel route
- [ ] Configure Zero Trust authentication
- [ ] Generate API key in Sure Finance
- [ ] Test transaction flow

---

*Last updated: 2026-01-18*
