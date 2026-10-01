# Ethical Self-Hosted LLM on Hetzner

Field notes from building a small, **locally inferred** language model stack for [Integrity Care Resources](https://my-integrity-hub.org/) — a 501(c)(3) nonprofit focused on trades training, veteran re-entry, and workforce development.

This is not a “jailbreak an LLM” guide. It is a record of how to put a model **on hardware you control**, keep inference **off third-party APIs**, serve a public website **without exposing the model**, and put **content policy in front of generation**.

Public site: [https://my-integrity-hub.org/](https://my-integrity-hub.org/)  
Nonprofit repo: [sdcastillo/integrity-care-solutions](https://github.com/sdcastillo/integrity-care-solutions)

---

## Why this exists

Most “AI products” send every prompt to a vendor. That is convenient. It is also a poor fit when you serve veterans, justice-involved people, or anyone whose questions should not become someone else’s training data.

The design goals for this build:

1. **Local inference** — the model runs on the VPS, not on a public chat vendor.
2. **Least exposure** — the public internet sees a static nonprofit website. The model is not the homepage.
3. **Identity-aware access** — admin and model traffic move over a private overlay (Tailscale), not open SSH on the public IP.
4. **Policy before generation** — the FastAPI layer applies a system prompt and a rating (G / PG / R) *before* tokens are produced.
5. **Separation of concerns** — HTML lives in `/var/www`. The app and models live under `/root`. Nginx does not have to proxy the public site through Python.

That last point is also an ethics point. If the LLM process dies, the nonprofit website should still load. People looking for job training should not get a `502 Bad Gateway` because a chatbot backend is down.

---

## Architecture (what we actually ran)

```
                         Cloudflare (proxy + TLS)
                                  |
                          public origin :80
                                  |
                    ┌────────────┐──────────────┐
                    │   Hetzner Ubuntu VPS       │
                    │                            │
                    │  Nginx  ──static──► /var/www/integritycare
                    │                            │
                    │  FastAPI / Uvicorn  :8000  │  (loopback or overlay only)
                    │         │                  │
                    │      Ollama :11434         │
                    │   dolphin-mistral:7b       │
                    │                            │
                    │  Tailscale overlay         │
                    │  hostname: milleniumfalcon │
                    └───────────────────────────┘
```

| Layer | Role | Safety reason |
|---|---|---|
| Cloudflare | TLS + hide origin from casual scanners | Public visitors never need raw SSH |
| Nginx `root` + `try_files` | Serves the nonprofit HTML | Site stays up if the LLM is off |
| FastAPI | Thin policy + `/chat` wrapper | Prompts are filtered *before* Ollama |
| Ollama | Local model runtime | Weights and chats stay on the box |
| Tailscale | Admin plane | No password SSH on the public internet |
| Fail2ban | SSH brute-force damping | Default-on for a public VPS |

---

## Ethical rules we encoded in software

The FastAPI app did not “trust the model.” It wrapped every request in an explicit policy:

- **G** — refuse violence, sexual content, illegal instructions, harm, heavy profanity. Redirect to a safe topic.
- **PG** — refuse explicit sexual content, graphic violence, crime how-tos, heavy profanity. Sanitize or redirect.
- **R / NC-17** — allow mature themes if requested; still refuse extreme gore, non-consensual content, and real-world crime instructions.

That is not perfect safety. No prompt is. It *is* better than dumping an unconstrained 7B chat model on a public URL and calling it a product.

Principles we treated as non-negotiable:

- Do not train on client conversations.
- Do not log full prompts to a world-readable file.
- Do not publish API keys, SSH keys, Auth0 secrets, or `.env` files.
- Do not expose Ollama (`:11434`) or Uvicorn (`:8000`) on `0.0.0.0` unless you have a reason and a firewall.
- Do not put HTML into `/etc/nginx/sites-available/`. That file is configuration, not a website.
- Do not make the nonprofit homepage depend on the model process.

---

## Directory contract (the lesson that cost us hours)

Two folders, two jobs:

```
/var/www/integritycare/     # public website (HTML, CSS, images)
/root/                      # backend: venv, FastAPI, Ollama models, logs
/etc/nginx/sites-available/ # Nginx server blocks only
```

Nginx config that serves the site **without** the LLM:

```nginx
server {
    listen 80;
    server_name example.org;

    root /var/www/integritycare;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

If you add `proxy_pass http://127.0.0.1:8000;` on `/` and Uvicorn is down, every visitor gets **502 Bad Gateway**. That is what happened when the public site and the chat app were glued together.

`proxy_pass` means: “do not read files from disk; forward this request to another process.” Use it only for paths that *must* hit the model (`/chat`, `/api`). Keep `/` static.

---

## Safe-ish baseline on a small VPS

These are operational habits, not an exploit cookbook.

1. Create the droplet (we used a 4 GB Helsinki instance). Enable the vendor firewall.
2. Install only what you need: `nginx`, `fail2ban`, Tailscale, Ollama, a Python venv.
3. Join Tailscale. Admin via the overlay hostname, not by opening port 22 to the world.
4. Pull a model you are willing to stand behind. We used `dolphin-mistral:7b` *behind* the policy layer above — the policy layer is the product, not the base model’s lack of refusal.
5. Bind Uvicorn to loopback (`127.0.0.1`) unless Tailscale or Nginx is the only ingress.
6. Confirm Ollama answers locally (`curl http://127.0.0.1:11434/api/tags`) before you wire FastAPI to it.
7. Put the nonprofit HTML in `/var/www/...`, `chown` it to `www-data`, restart Nginx, test `curl -I http://127.0.0.1`.
8. Put Cloudflare (or equivalent) in front of the domain. Origin stays a detail, not a marketing bullet.

### Binding the app (do this on purpose)

```bash
# Local-only — Nginx or an SSH/Tailscale tunnel reaches it
uvicorn main:app --host 127.0.0.1 --port 8000

# Overlay-visible — only if you understand who is on your tailnet
# uvicorn main:app --host 0.0.0.0 --port 8000
```

A process that “starts then immediately shuts down” under `nohup` is usually a missing dependency, a blocked startup hook (Ollama not running), or the parent shell dying. Prefer a systemd unit for anything you want alive after logout.

---

## What this is not

- Not a guide to removing safety from models.
- Not a guide for "super intelligence" or AGI
- Not a guide to attacking Hetzner, Nginx, or Tailscale.
- Not a claim that a 7B model is “aligned.”
- Not legal advice. If you serve protected populations, talk to counsel about data retention and consent.

---

## What we learned the hard way

| Symptom | Actual cause | Fix |
|---|---|---|
| `502 Bad Gateway` on the homepage | Nginx still `proxy_pass`-ing `/` to a dead Uvicorn | Serve `/` with `root` + `try_files` |
| Config “looks fine” but 502 persists | HTML pasted into `sites-available`, or leftover enabled site | One server block, `nginx -t`, restart, read `error.log` |
| `root /var/www/integrityhub` 404s | Files lived in `/var/www/integritycare` | Config `root` must match the real directory |
| Uvicorn “Waiting for application startup” then dies | Ollama down, or background job killed | Check `ollama serve`, run foreground once, then systemd |
| Site works on localhost, not on Tailscale name | App bound to `127.0.0.1` only, or DNS/health warnings on the client | Decide loopback vs overlay on purpose |

Read the log. Guessing wastes more time than `tail -n 50 /var/log/nginx/error.log`.

---

## Minimal FastAPI shape (policy in front of Ollama)

Sketch only — keep secrets out of git.

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from pydantic import BaseModel
import requests

app = FastAPI()

PROMPTS = {
    "G": "Refuse violence, sexual content, illegal acts, harm, and profanity. Redirect to a safe topic.",
    "PG": "Refuse explicit sexual content, graphic violence, crime instructions, and heavy profanity.",
    "R": "Allow mature themes if asked. Refuse extreme gore, non-consensual content, and real crime instructions.",
}

class ChatRequest(BaseModel):
    message: str
    rating: str = "PG"

@app.post("/chat")
def chat(req: ChatRequest):
    system = PROMPTS.get(req.rating.upper(), PROMPTS["PG"])
    payload = {
        "model": "dolphin-mistral:7b",
        "messages": [
            {"role": "system", "content": system},
            {"role": "user", "content": req.message},
        ],
        "stream": False,
    }
    try:
        res = requests.post("http://127.0.0.1:11434/api/chat", json=payload, timeout=60)
        res.raise_for_status()
        return {"response": res.json()["message"]["content"]}
    except Exception as exc:
        return JSONResponse({"error": "upstream model unavailable"}, status_code=503)
```

The important line is the **system prompt chosen by rating**, not the model name.

---

## Knowledge Pays Most

Integrity Care’s public work is trades training — electrical, machining, construction, logistics, employability. The LLM, if used at all, is a helper behind that mission: explain a syllabus, draft a polite email, practice interview language. It is not the product.

If you copy this stack, copy the **separation** and the **policy layer**. Leave the jailbreaks on the cutting-room floor.

— Samuel Castillo, 2026
