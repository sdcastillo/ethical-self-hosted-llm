---
layout: default
title: Ethical Self-Hosted LLM
description: Field notes from a Hetzner, Tailscale, Nginx, and Ollama build that keeps nonprofit inference on hardware you control.
samwiki: true
---

<section class="sw-lede" aria-labelledby="stack-title">
  <div class="sw-lede-copy">
    <h2 id="stack-title">Inference on the box you administer</h2>
    <p>These notes record the stack built for <a href="https://my-integrity-hub.org/">Integrity Care Resources</a>, a 501(c)(3) that trains people for the trades and for veteran re-entry. The model runs on a Hetzner Ubuntu VPS. Prompts stay on that machine. They are not sent to a public chat API, and this is not a guide to stripping refusals out of a model.</p>
    <p>The public internet only needs the nonprofit website. Cloudflare proxies the domain and terminates TLS. Nginx serves static HTML from <code>/var/www/integritycare</code> with <code>root</code> and <code>try_files</code>. The language model is not the homepage. If Ollama or Uvicorn stops, the job-training pages should still return 200.</p>
    <p>Inference sits behind that split. FastAPI on <code>127.0.0.1:8000</code> is the only application that should call Ollama on <code>127.0.0.1:11434</code>. The weights we ran were <code>dolphin-mistral:7b</code>. Admin SSH goes over Tailscale to the overlay hostname <code>milleniumfalcon</code>, not to port 22 on the public address. Fail2ban stays on for the VPS. HTML, the Python app, and Nginx config live in three different places: <code>/var/www/integritycare</code>, <code>/root</code>, and <code>/etc/nginx/sites-available</code>.</p>
    <p>Two mistakes produced the outages. Pasting the website into a file under <code>sites-available</code>, or pointing <code>location /</code> at <code>proxy_pass http://127.0.0.1:8000</code> while Uvicorn was down, turned the whole origin into 502 Bad Gateway. <code>proxy_pass</code> belongs on <code>/chat</code> and <code>/api</code> only. Before any tokens are generated, the API picks a system prompt from a rating. G refuses violence, sexual content, illegal instructions, and harm. PG refuses explicit sexual content, graphic violence, and crime how-tos. R still refuses extreme gore, non-consensual content, and real-world crime instructions. Client conversations are not training data, prompt logs are not world-readable, and secrets do not go in git.</p>
  </div>
  <aside class="sw-find" aria-labelledby="layers-title">
    <h2 id="layers-title">What stays separated</h2>
    <ul>
      <li><strong>Cloudflare</strong> TLS and a hidden origin. Visitors never need the raw VPS address.</li>
      <li><strong>Nginx</strong> Static <code>root</code> plus <code>try_files</code>. The site does not depend on the model process.</li>
      <li><strong>FastAPI</strong> Loopback on port 8000. A rating selects the system prompt before generation.</li>
      <li><strong>Ollama</strong> Loopback on port 11434. Weights and chats stay on the box.</li>
      <li><strong>Tailscale</strong> Admin plane. No password SSH on the public IP.</li>
    </ul>
  </aside>
</section>
