# Configuration DNS (Local)

Pour accéder aux services de la sandbox via des noms de domaine (ex: `*.okdp.dev-sandbox`), un serveur DNS local est déployé dans le cluster sur le port **30053** (UDP).

Cette configuration permet de ne pas avoir à modifier le fichier `/etc/hosts` pour chaque nouveau service.

## Configuration Resolver (macOS / Linux)

Créez (ou modifiez) le fichier `/etc/resolver/okdp.dev-sandbox` pour rediriger toutes les requêtes du domaine `.okdp.dev-sandbox` vers le serveur DNS local.

**Fichier : `/etc/resolver/okdp.dev-sandbox`**

```text
nameserver 127.0.0.1
port 30053
```

*(Note : Sous macOS, assurez-vous que le répertoire `/etc/resolver` existe via `sudo mkdir -p /etc/resolver`).*

## Vérification

Testez la résolution d'un sous-domaine quelconque :

```bash
ping test.okdp.dev-sandbox
# Résultat attendu :
# PING test.okdp.dev-sandbox (127.0.0.1): 56 data bytes
# 64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.059 ms
```

Si la résolution fonctionne, vous pouvez accéder aux services via HTTPS (ex: `https://kubauth.okdp.dev-sandbox`).
