# Nevatun

**Nevatun** est un fork stabilisé de [browse](https://github.com/anomalyco/browse) (MIT) : un relais local et une extension Chromium qui permettent à un agent (OpenCode, Claude Code, Codex, Cursor…) de piloter un **vrai Chrome** — ses onglets, ses sessions, ses extensions — sans navigateur headless.

> Base upstream : `anomalyco/browse` **v0.0.17** (`ac696c8`). Les changements de ce fork sont listés dans [`NOTICE`](NOTICE).

---

## Sommaire

1. [Pourquoi ce fork](#pourquoi-ce-fork)
2. [Architecture](#architecture)
3. [Prérequis](#prérequis)
4. [Installation](#installation)
5. [Utilisation](#utilisation)
6. [Configuration](#configuration)
7. [Développement](#développement)
8. [Sécurité](#sécurité)
9. [Licence](#licence)

## Pourquoi ce fork

browse fonctionne bien, mais la liaison entre le relais et l'extension pouvait se couper quand Chrome endormait le service worker MV3 de l'extension. Ce fork corrige la cause et durcit la reprise :

- **Keepalive MV3** : le heartbeat appelle une API `chrome.*`, ce qui réarme le timer d'inactivité du service worker.
- **Attente de reconnexion** : un appel patiente pendant la fenêtre de reconnexion de l'extension au lieu d'échouer immédiatement.
- **Timeout de ping aligné** (10 s → 30 s) sur la période d'alarme de l'extension.
- **Mode distant derrière un proxy local** : l'upgrade WebSocket `/extension` ne subit plus le contrôle `Host` (l'`Origin` et l'appairage restent), ce qui permet un accès via `tailscale serve`.

## Architecture

```
Agent (OpenCode, ...)
   │  MCP stdio ou CLI
   ▼
Relais Nevatun  (127.0.0.1:19988, machine de l'agent)
   │  WebSocket /extension
   ▼
Extension Chromium  (Chrome de l'utilisateur, éventuellement sur une autre machine)
   │  chrome.debugger (CDP)
   ▼
Onglets réels
```

- Le relais vit **à côté de l'agent** : le MCP/CLI parle au relais en `127.0.0.1:19988`.
- L'extension peut être sur une autre machine ; le réseau doit être privé (Tailscale, WireGuard, LAN de confiance). La liaison n'est **pas chiffrée** par Nevatun.
- Plusieurs agents et onglets sont supportés ; les onglets ouverts par un agent sont regroupés et suivis.

## Prérequis

- **Node.js ≥ 22.19** (ou Bun) pour le CLI et le relais.
- **Chrome / Chromium / Edge / Brave** avec l'extension Nevatun chargée.
- Pour le mode distant : un réseau privé (Tailscale recommandé).

## Installation

### 1. Sources et build

```sh
git clone <votre-dépôt> Nevatun
cd Nevatun
bun install
bun run build                              # extension → dist/
cd cli && bun install && bun run build     # CLI → cli/dist/cli.mjs
```

### 2. Relais

```sh
# local uniquement (127.0.0.1)
node cli/dist/cli.mjs serve

# ou, pour un navigateur sur une autre machine
node cli/dist/cli.mjs set hostname 0.0.0.0
node cli/dist/cli.mjs serve
```

Exemple de service persistant (systemd) :

```ini
[Unit]
Description=Nevatun relay
After=network-online.target

[Service]
User=<user>
ExecStart=/usr/bin/node /opt/Nevatun/cli/dist/cli.mjs serve
Restart=always

[Install]
WantedBy=multi-user.target
```

### 3. Extension

1. `chrome://extensions` → retirez l'extension **browse** du Chrome Web Store.
2. Activez **Mode développeur**.
3. **Charger l'extension non empaquetée** → sélectionnez le dossier `dist/`.

### 4. Appairage

```sh
node cli/dist/cli.mjs pair
```

Dans les **Options** de l'extension → *Advanced* → **Pair with a code** :

- **Adresse du relais** : `<hôte>:19988`
- **Code** : celui affiché (usage unique, valable 10 minutes)

## Utilisation

### CLI

```sh
nevatun status
nevatun pair --list
nevatun session new ma-session
nevatun execute -s ma-session --file script.js
```

Le script `execute` reçoit `page`, `browser`, `context`, `snapshot()`, `ref()` et `handoff()`. Exemple :

```js
await page.goto("https://example.com", { waitUntil: "domcontentloaded" });
return { url: page.url(), title: await page.title() };
```

### MCP (agent)

Déclarez le serveur dans la configuration de l'agent. Exemple pour OpenCode :

```jsonc
{
  "mcp": {
    "nevatun": {
      "type": "local",
      "command": ["/chemin/vers/nevatun", "mcp"],
      "enabled": true
    }
  }
}
```

L'agent dispose alors des outils natifs (`navigate`, `snapshot`, `click`, `fill`, `screenshot`, `network`, `recording`, `tabs`, `status`, `execute`…).

## Configuration

- `~/.config/browse/config.json` : `{"hostname":"127.0.0.1"}` ou `"0.0.0.0"`.
- Variables d'environnement (préfixe historique conservé) :
  - `OPENCODE_BROWSER_PING_TIMEOUT_MS` (défaut `30000`)
  - `OPENCODE_BROWSER_EXTENSION_TIMEOUT_MS` (défaut `30000`)
  - `OPENCODE_BROWSER_PORT`, `OPENCODE_BROWSER_TOOLS`, `OPENCODE_BROWSER_HUMAN_INPUT`, etc.
- Fichiers locaux : `~/.config/browse`, `~/.local/share/browse`, `~/.local/state/browse`, `~/.cache/browse`.
- Journaux : `~/.local/state/browse/logs/relay.log`.

## Développement

```sh
bun run typecheck
bun test ./test            # extension / cœur
cd cli && bun test         # CLI / relais
```

Le code reste compatible avec l'upstream ; les patches de ce fork sont volontairement minimaux et documentés dans `NOTICE`.

## Sécurité

- **Aucun secret dans le dépôt** : tokens, clés privées et codes d'appairage ne doivent jamais être commités.
- Le token du relais et les appairages vivent dans `~/.local/share/browse/` avec des permissions restreintes ; ne les copiez pas dans le dépôt.
- L'extension est authentifiée par appairage (code à usage unique, clé propre à chaque navigateur) et par `Origin` ; le relais refuse les origines ou agents non appairés.
- En mode distant, utilisez un réseau privé (Tailscale/WireGuard) : Nevatun n'ajoute pas de chiffrement.
- Révoquez ce qui a été exposé : `nevatun pair --revoke <navigateur>` et rotation des clés d'accès.

## Licence

MIT — voir [`LICENSE`](LICENSE). Nevatun est un fork de browse ; voir [`NOTICE`](NOTICE) pour l'attribution et la liste des modifications.
