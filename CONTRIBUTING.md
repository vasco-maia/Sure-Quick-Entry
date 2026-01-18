# Contributing

Thanks for considering contributing! I'm primarily looking for **security feedback** and **code review**.

## What I'm Looking For

### 🔒 Security Feedback (Most Wanted!)

- Point out vulnerabilities or weaknesses
- Suggest better approaches
- Review the nginx configuration
- Question my assumptions
- Share resources I should read

**How to contribute:**
1. Open an issue with the "Security Feedback" label
2. Start a GitHub Discussion
3. For sensitive issues, see [SECURITY.md](SECURITY.md)

### 👀 Code Review

- Is there a cleaner way to do something?
- Am I misusing an API?
- Are there edge cases I'm missing?
- JavaScript/React best practices I'm violating?

### 📝 Documentation

- Clarify confusing sections
- Fix typos
- Add missing information
- Improve the threat model

---

## What I'm NOT Looking For

To keep this project simple:

- ❌ Major rewrites in different frameworks
- ❌ Feature bloat (more features = more attack surface)
- ❌ Build systems / complex tooling
- ❌ TypeScript conversion (keeping it simple)
- ❌ Backend changes (that's Sure Finance's domain)

---

## How to Submit Feedback

### Option 1: GitHub Issue

Best for: Specific bugs, concrete suggestions, security concerns (non-sensitive)

1. Check if a similar issue exists
2. Use an issue template if applicable
3. Be specific about what you're suggesting

### Option 2: GitHub Discussion

Best for: Questions, general feedback, "have you considered..." conversations

### Option 3: Pull Request

Best for: Documentation fixes, small improvements

**Before submitting a PR:**
- Keep changes small and focused
- Explain what and why in the description
- Don't break the single-file simplicity

---

## Code Style

This is a single HTML file with embedded JavaScript. I'm keeping it simple:

- No build step
- No external dependencies (except CDN)
- Readable over clever
- Comments where logic isn't obvious

If you suggest code changes, please keep this philosophy in mind.

---

## Questions?

Open an issue or discussion. I'm happy to chat!

---

## Recognition

Contributors who provide valuable feedback will be acknowledged in:
- [SECURITY.md](SECURITY.md) - Security contributors
- [README.md](README.md) - General acknowledgments

Let me know if you'd prefer to remain anonymous.
