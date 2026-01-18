# Sure-Quick-Entry

A simple PWA for quickly logging transactions to [Sure Finance](https://github.com/we-promise/sure) on mobile.

> **🔒 Looking for Security Feedback**: I built this for personal use but lack deep security expertise. If you spot issues or have suggestions, please share! See [SECURITY.md](SECURITY.md).

---

## Why I Built This

Sure Finance is great for personal finance tracking, but entering transactions on mobile was slow. I wanted something I could open, tap a few times, and be done in seconds while standing in line at a store.

This is a single HTML file that runs as a PWA, proxied through Nginx to talk to my self-hosted Sure Finance instance.

**I'm not a security expert** — I'm a homelab enthusiast learning as I go. That's why I'm sharing this publicly: to get feedback from people who know more than me.

---

## What It Does

```
┌─────────────────┐         ┌──────────────────┐
│  Phone Browser  │ ──────▶ │  Cloudflare      │
│  (PWA)          │  HTTPS  │  Tunnel + Zero   │
└─────────────────┘         │  Trust Auth      │
                            └────────┬─────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │  Nginx Container │
                            │  (sure-quick)    │
                            └────────┬─────────┘
                                     │
                  ┌──────────────────┴──────────────────┐
                  │                                     │
                  ▼                                     ▼
         ┌─────────────────┐                  ┌─────────────────┐
         │  Static HTML    │                  │  Proxy /api/*   │
         │  (React PWA)    │                  │  requests       │
         └─────────────────┘                  └────────┬────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │  Sure Finance   │
                                              │  (Rails API)    │
                                              └─────────────────┘
```

1. User opens PWA on phone
2. Cloudflare Zero Trust checks authentication
3. Nginx serves the static HTML
4. API calls go through Nginx proxy to Sure Finance
5. Transaction is saved

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| Frontend | React 18, Tailwind CSS (both from CDN) |
| Transpiler | Babel (in-browser) |
| Server | Nginx Alpine |
| Container | Docker |
| Backend | Sure Finance (Ruby on Rails) |
| Auth | API Key via `X-API-Key` header |
| Access Control | Cloudflare Zero Trust |

---

## Project Structure

```
sure-quick-entry/
├── README.md
├── SECURITY.md
├── CONTRIBUTING.md
├── LICENSE
├── docker-compose.yml
├── src/
│   ├── index.html        # The entire app (single file)
│   ├── nginx.conf        # Proxy configuration
│   └── manifest.json     # PWA manifest
├── docs/
│   ├── ARCHITECTURE.md
│   └── THREAT_MODEL.md
└── .github/
    └── ISSUE_TEMPLATE/
        └── security-feedback.md
```

---

## Security Considerations (What I Know)

I've documented what I understand about the security implications:

| Area | Current Approach | My Concern Level |
|------|------------------|------------------|
| API Key Storage | localStorage | ⚠️ Medium - Is this okay? |
| CDN Dependencies | Unpkg, Tailwind CDN | ⚠️ Medium - Supply chain? |
| In-browser Babel | Runtime transpilation | 🤔 Low - Performance only? |
| Transport | HTTPS via Cloudflare | ✅ Seems fine |
| Access Control | Cloudflare Zero Trust | ✅ Seems fine |

**I'd love input on:**
- Is localStorage acceptable for API keys in this context?
- Should I add Subresource Integrity (SRI) hashes for CDN scripts?
- Are there XSS vectors I'm missing?
- Is my nginx proxy config secure?
- What else am I not thinking about?

See [docs/THREAT_MODEL.md](docs/THREAT_MODEL.md) for my analysis (and gaps).

---

## Quick Start

### Prerequisites

- Docker & Docker Compose
- Running Sure Finance instance
- API Key from Sure Finance (Settings → API Keys)
- Cloudflare Tunnel (recommended for external access)

### Deployment

1. Clone this repo:
```bash
git clone https://github.com/YOUR_USERNAME/sure-quick-entry.git
cd sure-quick-entry
```

2. Copy source files to your deployment location:
```bash
mkdir -p /srv/docker/sure-quick
cp src/* /srv/docker/sure-quick/
cp docker-compose.yml /srv/docker/sure-quick/
```

3. Add an app icon (192x192 PNG):
```bash
# Create or copy your icon
cp your-icon.png /srv/docker/sure-quick/icon-192.png
```

4. Update nginx.conf with your Sure Finance URL:
```nginx
proxy_pass https://YOUR_FINANCE_URL/api/;
proxy_set_header Host YOUR_FINANCE_URL;
```

5. Start the container:
```bash
cd /srv/docker/sure-quick
docker compose up -d
```

6. (Optional) Configure Cloudflare Tunnel:
```yaml
- hostname: quick.yourdomain.com
  service: http://sure-quick:80
```

7. Open the app and paste your API key when prompted

---

## API Endpoints Used

The app calls these Sure Finance endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/accounts` | GET | List accounts with balances |
| `/api/v1/categories` | GET | List expense/income categories |
| `/api/v1/merchants` | GET | List merchants |
| `/api/v1/tags` | GET | List tags |
| `/api/v1/transactions` | POST | Create new transaction |

### Transaction Payload

```json
{
  "transaction": {
    "account_id": "uuid",
    "date": "2026-01-18",
    "amount": 25.50,
    "name": "Coffee",
    "nature": "expense",
    "category_id": "uuid|null",
    "merchant_id": "uuid|null",
    "tag_ids": ["uuid"]|null,
    "notes": "string|null"
  }
}
```

---

## Screenshots

*Coming soon - or submit a PR!*

---

## Contributing

I'm primarily looking for:

1. **🔒 Security feedback** - Please open an issue or discussion!
2. **👀 Code review** - Is there a better/safer way to do something?
3. **💡 Suggestions** - Improvements, gotchas I should know about

Not looking for:
- Major rewrites or framework changes (keeping it simple)
- Feature bloat

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## Credits & Acknowledgments

- **[Sure Finance](https://github.com/we-promise/sure)** - The backend this connects to
- **[Maybe Finance](https://github.com/maybe-finance/maybe)** - Original project (archived) that Sure is forked from
- **Claude (Anthropic)** - Helped me build and document this
- **The homelab & self-hosting community** - For endless inspiration

---

## Why Share This?

I'm sharing this to:
1. **Get security feedback** from people smarter than me
2. **Help others** who might want something similar
3. **Learn** from the community

This is a personal project I hacked together for my own use. It works for me, but I make no claims about it being production-ready or secure.

---

## License

MIT License - See [LICENSE](LICENSE)

---

## Disclaimer

⚠️ **Use at your own risk.**

This is a personal project with no warranty. I am not responsible for any data loss, security breaches, or other issues. If you deploy this, you're responsible for securing it properly.

If you find a security vulnerability, please see [SECURITY.md](SECURITY.md).
