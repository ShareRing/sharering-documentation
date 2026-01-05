This is a test doc, please ignore it.

# ShareRing Me Modules Developer Guide (V2)

This guide is for **external developers** who want to build a **Me Module** for the ShareRing Me app.

- A **Me Module** is a **web application** that runs inside the ShareRing Me mobile app (in an embedded WebView).
- A Me Module communicates with the ShareRing Me app **only** via a **message bridge** (`postMessage`).
- This document covers the **V2 Me Module API**.

---

## What you build (high-level)

- A static web app (React/Vue/Svelte/Vanilla JS—your choice)
- Hosted at a public URL (typically `https://...`)
- With a required `manifest.json` at the **domain root**
- Optionally packaged for **offline caching** using a zip bundle

---

## Quickstart (recommended scaffolding)

You can use any stack. For best results (TypeScript + fast iteration + predictable build output) use **Vite + React + TypeScript**.
