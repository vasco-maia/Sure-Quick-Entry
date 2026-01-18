# Threat Model

> ⚠️ **Disclaimer**: I'm not a security professional. This is my best attempt at documenting security considerations. Please help me improve this!

## Overview

Sure Quick Entry is a Progressive Web App for logging financial transactions. It handles:
- API credentials (stored locally)
- Financial transaction data (passed through to Sure Finance)
- Account balances (displayed from API)

---

## System Context

```
┌─────────────────────────────────────────────────────────────────────┐
│                          TRUST BOUNDARY                              │
│  ┌──────────────┐                                                   │
│  │   Attacker   │                                                   │
│  └──────┬───────┘                                                   │
│         │ Must bypass:                                              │
│         │ 1. Cloudflare Zero Trust (authentication)                 │
│         │ 2. HTTPS encryption                                       │
│         ▼                                                           │
├─────────────────────────────────────────────────────────────────────┤
│                        PROTECTED ZONE                                │
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   Browser    │────▶│    Nginx     │────▶│ Sure Finance │        │
│  │   (PWA)      │     │   Proxy      │     │    API       │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│         │                                                           │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │ localStorage │ ← API Key stored here                             │
│  └──────────────┘                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Assets (What We're Protecting)

| Asset | Sensitivity | Location |
|-------|-------------|----------|
| API Key | High | Browser localStorage |
| Account Balances | Medium | In-memory (from API) |
| Transaction History | Medium | Sure Finance (not stored locally) |
| Transaction Input | Low-Medium | In-memory during entry |

---

## Threat Actors

| Actor | Capability | Motivation |
|-------|------------|------------|
| **Random Internet Attacker** | Low - blocked by Zero Trust | Financial data theft |
| **Compromised CDN** | High - can inject arbitrary JS | Credential harvesting |
| **Malicious Browser Extension** | High - full DOM access | Data theft |
| **Physical Device Access** | High - full localStorage access | Financial data |
| **Network Attacker (MITM)** | Low - HTTPS in place | Credential theft |

---

## Attack Vectors & Mitigations

### 1. Unauthorized Access

| Attack | Risk | Mitigation | Status |
|--------|------|------------|--------|
| Direct URL access | Medium | Cloudflare Zero Trust | ✅ Implemented |
| Brute force auth | Low | Zero Trust rate limiting | ✅ Cloudflare handles |
| Session hijacking | Low | Zero Trust sessions | ✅ Cloudflare handles |

### 2. API Key Theft

| Attack | Risk | Mitigation | Status |
|--------|------|------------|--------|
| XSS to steal localStorage | Medium | React escaping, Zero Trust barrier | ⚠️ Partial |
| Malicious browser extension | Medium | None (user responsibility) | ❌ Not mitigated |
| Physical device access | Medium | Device security (user responsibility) | ❌ Not mitigated |
| CDN compromise (supply chain) | Medium | Could add SRI hashes | ⚠️ Not implemented |

**My Analysis**: The API key in localStorage is the biggest concern. However:
- Attacker must first bypass Zero Trust auth
- API key can be revoked if compromised
- This is a single-user personal app

**Question for reviewers**: Is there a better approach that doesn't add significant complexity?

### 3. Cross-Site Scripting (XSS)

| Vector | Risk | Mitigation | Status |
|--------|------|------------|--------|
| Reflected XSS | Low | No URL parameter handling | ✅ N/A |
| Stored XSS | Low | No user content stored/displayed | ✅ N/A |
| DOM XSS | Low | React handles escaping | ⚠️ Assumed safe |
| CDN script injection | Medium | No SRI hashes | ❌ Not mitigated |

**My Analysis**: The app doesn't display user-generated content (only API responses). React should handle escaping. But the CDN scripts are a concern.

**Question for reviewers**: Am I missing XSS vectors? Should I be more worried?

### 4. Supply Chain Attacks

| Component | Risk | Mitigation | Status |
|-----------|------|------------|--------|
| React (unpkg) | Medium | Could add SRI | ❌ Not implemented |
| ReactDOM (unpkg) | Medium | Could add SRI | ❌ Not implemented |
| Babel (unpkg) | Medium | Could add SRI | ❌ Not implemented |
| Tailwind (CDN) | Low | Could add SRI | ❌ Not implemented |

**Potential Fix**:
```html
<script 
  src="https://unpkg.com/react@18/umd/react.production.min.js"
  integrity="sha384-[hash]"
  crossorigin="anonymous"
></script>
```

**Question for reviewers**: Is this worth doing? Should I just bundle everything locally instead?

### 5. Man-in-the-Middle (MITM)

| Attack | Risk | Mitigation | Status |
|--------|------|------------|--------|
| Network interception | Very Low | HTTPS via Cloudflare | ✅ Implemented |
| SSL stripping | Very Low | HSTS via Cloudflare | ✅ Cloudflare handles |
| DNS spoofing | Very Low | Cloudflare DNS | ✅ Cloudflare handles |

**My Analysis**: HTTPS everywhere should handle this. Cloudflare Tunnel means no ports exposed on my network.

### 6. Nginx Misconfiguration

| Issue | Risk | Current Status |
|-------|------|----------------|
| Overly permissive proxy | Medium | Only `/api/*` proxied |
| Missing security headers | Low-Medium | Not configured |
| Directory traversal | Low | try_files configured |

**Current nginx.conf security posture**:
```nginx
server {
    listen 80;
    
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
    
    location /api/ {
        proxy_pass https://finance.mayavasko.uk/api/;
        # ... proxy headers ...
    }
}
```

**Missing (should add?)**:
```nginx
# Security headers
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

**Question for reviewers**: What else should be in the nginx config?

---

## Data Flow Security

```
User Input → React State → API Call → Nginx Proxy → Sure Finance
                ↓
        localStorage (API Key only)
```

| Stage | Data | Encryption | Persistence |
|-------|------|------------|-------------|
| User Input | Transaction details | None (in-memory) | None |
| React State | Form data | None (in-memory) | None |
| API Call | Transaction + API Key | HTTPS | None |
| localStorage | API Key | None | Persistent |

---

## Risk Summary

| Risk | Likelihood | Impact | Overall | Mitigation Priority |
|------|------------|--------|---------|---------------------|
| CDN compromise | Low | High | Medium | Should add SRI |
| API key theft via XSS | Low | High | Medium | React escaping + Zero Trust |
| Physical device theft | Low | Medium | Low | User responsibility |
| Zero Trust bypass | Very Low | High | Low | Trust Cloudflare |
| Nginx misconfiguration | Low | Medium | Low | Review config |

---

## Recommendations (For Myself)

### High Priority
1. [ ] Add SRI hashes to CDN scripts
2. [ ] Add security headers to nginx
3. [ ] Document API key rotation procedure

### Medium Priority
1. [ ] Consider bundling dependencies locally
2. [ ] Add Content-Security-Policy header
3. [ ] Review nginx proxy configuration

### Low Priority
1. [ ] Add rate limiting (if not relying on Zero Trust)
2. [ ] Implement API key expiration/rotation reminder

---

## Open Questions

I'd appreciate input on:

1. **Is localStorage acceptable here?** Given Zero Trust is in front?

2. **SRI vs Local Bundling?** Which is better for security/simplicity trade-off?

3. **What CSP would work?** With CDN scripts and inline styles?

4. **Am I overthinking this?** It's a personal single-user app.

5. **Am I underthinking this?** What am I missing?

---

## References

Resources I've read (or should read):
- [OWASP Top 10](https://owasp.org/Top10/)
- [OWASP Cheat Sheet - XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [MDN - Subresource Integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)
- [Cloudflare Zero Trust Docs](https://developers.cloudflare.com/cloudflare-one/)

---

*Last updated: 2026-01-18*
