# Security Policy

## ⚠️ Important Disclaimer

I'm not a security expert. I built this for personal use and I'm sharing it to get feedback from people who know more than me. **Please help me make this better!**

---

## Reporting Issues

### For Security Vulnerabilities

If you find something that could be exploited:

1. **Please don't open a public issue** if it's sensitive
2. Instead, use GitHub's [private vulnerability reporting](../../security/advisories/new)
3. Or email me directly (add your email here)

I'll do my best to respond within 48 hours and work on a fix.

### For General Security Feedback

If it's not an active vulnerability but more of a "you should probably do X differently":

- **Open an issue!** I want to learn.
- Or start a **GitHub Discussion**
- Use the "Security Feedback" issue template

---

## What I Think I Know (Please Correct Me)

### Things I'm Doing

| Practice | Why I Think It's Okay | Am I Wrong? |
|----------|----------------------|-------------|
| localStorage for API key | Single-user app, behind Zero Trust | Maybe? |
| CDN scripts (React, Tailwind) | Simpler deployment | Should add SRI? |
| Nginx proxy for API | Same-origin for browser | Config issues? |
| HTTPS everywhere | Via Cloudflare Tunnel | Seems fine? |
| Zero Trust access | Requires auth before seeing app | Good layer? |

### Things I'm Worried About

| Concern | My Current Thinking | Need Input |
|---------|---------------------|------------|
| XSS attacks | React escapes by default, but... | What am I missing? |
| API key theft | localStorage + Zero Trust = okay? | Better options? |
| Supply chain (CDNs) | Trust unpkg/cdnjs? | SRI hashes? |
| Nginx misconfiguration | Copied from examples | Review needed |

---

## Security Layers (As I Understand Them)

```
Layer 1: Cloudflare Zero Trust
├── Must authenticate to reach the app at all
├── Email/SSO verification required
└── Blocks unauthorized access

Layer 2: HTTPS
├── Cloudflare Tunnel encrypts traffic
├── No exposed ports on my network
└── CG-NAT adds obscurity (not security)

Layer 3: API Key
├── Scoped to Sure Finance account
├── Can be revoked if compromised
└── Stored in localStorage (concern?)

Layer 4: Application
├── React sanitizes output (XSS protection?)
├── No user-generated HTML rendering
└── All inputs go through controlled components
```

**Question**: Is this "defense in depth" or am I fooling myself?

---

## Known Limitations

Things I know aren't ideal but accepted for simplicity:

### 1. localStorage for API Key

**What**: The API key is stored in browser localStorage.

**Why I did it**: Simplest approach for a single-user PWA.

**Risks I understand**:
- XSS could steal it
- Browser extensions could read it
- Persists until manually cleared

**Mitigations**:
- Zero Trust means attacker needs my auth first
- API key can be revoked
- Single-user app (I'm the only user)

**Better alternatives?** I'd love to know.

### 2. CDN Dependencies

**What**: React, ReactDOM, Babel, and Tailwind loaded from CDNs.

**Why I did it**: Single HTML file, no build step.

**Risks I understand**:
- CDN compromise could inject malicious code
- No integrity verification currently

**Potential fix**: Add SRI hashes. Should I?

```html
<!-- Could add integrity attributes like this? -->
<script 
  src="https://unpkg.com/react@18/umd/react.production.min.js"
  integrity="sha384-XXXXX"
  crossorigin="anonymous"
></script>
```

### 3. In-Browser Babel

**What**: JSX transpiled at runtime in browser.

**Why I did it**: Avoid build step, single file simplicity.

**Risks I understand**:
- Performance hit (acceptable for small app)
- Larger payload

**Security risk?** I don't think so, but correct me if wrong.

### 4. No CSP Headers

**What**: Not setting Content-Security-Policy headers.

**Why**: Didn't think about it / not sure how with CDN scripts.

**Should I add**? What would a good CSP look like for this setup?

---

## Security Checklist (For Deployers)

If you deploy this yourself, please:

- [ ] Use HTTPS (Cloudflare Tunnel, reverse proxy, etc.)
- [ ] Add authentication layer (Zero Trust, Authelia, etc.)
- [ ] Use a dedicated API key (not your main one)
- [ ] Limit API key permissions if possible
- [ ] Review nginx.conf for your environment
- [ ] Consider adding SRI hashes to CDN scripts
- [ ] Don't expose this directly to the internet without auth

---

## Questions I Have

1. **Is localStorage fundamentally unsafe for API keys?** Even with Zero Trust in front?

2. **What XSS vectors exist in this app?** I'm using React but loading external scripts.

3. **Should I bundle dependencies instead of using CDNs?** Is the trade-off worth it?

4. **Is my nginx proxy config secure?** Are there headers I should add/remove?

5. **What am I not even thinking about?** Unknown unknowns scare me.

---

## Acknowledgments

Thanks to anyone who takes time to review this and provide feedback. I'm here to learn!

If you help improve the security of this project, I'll add you here (with your permission):

### Security Contributors

*Be the first!*
