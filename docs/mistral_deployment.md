# Déploiement Blender-MCP pour Mistral et n8n

Ce guide explique comment exposer le serveur Blender-MCP sur Internet de façon sécurisée afin qu’il soit accessible depuis Mistral AI (Le Chat, API ou Workflows) ainsi que depuis un workflow n8n, tout en conservant Blender sur votre Mac et le serveur réseau sur votre NAS Synology.

## 1. Architecture cible

- **Mac** : Blender + addon `blender-mcp` qui ouvre un socket (par défaut `9876`) en écoute sur le Mac.
- **NAS Synology (Docker)** : conteneur `blender-mcp` qui parle au socket Blender via le réseau local et expose une API MCP sur HTTPS vers Internet.
- **Clients** : Mistral AI (site web ou API) et n8n se connectent à l’URL publique sécurisée du NAS.

```
[Mistral / n8n] <--HTTPS--> [Reverse proxy Synology] <--LAN--> [Conteneur MCP] <--LAN--> [Blender sur Mac]
```

## 2. Pré-requis réseau

1. **Nom de domaine** : enregistrez un sous-domaine dédié (ex. `mcp.mondomaine.tld`) pointant vers votre IP publique.
2. **Redirection de port** : sur la box Internet, redirigez le port 443 vers l’IP locale du NAS (port 443 ou 8443 du reverse proxy).
3. **VPN ou tunnel chiffré** *(recommandé)* : si possible, préférez un accès via Tailscale, Cloudflare Tunnel ou Synology QuickConnect pour éviter l’ouverture directe du port 443.

## 3. Reverse proxy Synology sécurisé

1. Dans DSM > Portail des applications, créez un reverse proxy :
   - Source : `https://mcp.mondomaine.tld` → Port 443.
   - Destination : `http://127.0.0.1:3000` (port interne du conteneur MCP exposé sur l’hôte).
2. Certificat TLS : générez un certificat Let’s Encrypt depuis DSM et associez-le au reverse proxy.
3. **Filtrage IP** : si vous connaissez les IP de vos clients (ex. serveur n8n), limitez l’accès aux plages autorisées.
4. **Auth. additionnelle** : activez au choix :
   - Authentification de base (login/mot de passe fort).
   - Jeton OAuth2/Access Token via un portail WAF (Synology « Application Portal » ou Nginx Proxy Manager).
5. Activez HTTP/2 et HSTS, désactivez les TLS obsolètes.

## 4. Conteneur Docker Blender-MCP

Dans le dossier du projet sur le NAS (`/volume1/docker/blender-mcp` par exemple) copiez les sources puis créez les fichiers suivants.

### `deploy/docker-compose.mistral.yml`

```yaml
services:
  blender-mcp:
    build:
      context: ..
      dockerfile: deploy/Dockerfile
    container_name: blender-mcp
    environment:
      # Adresse du Mac sur le LAN
      BLENDER_HOST: 192.168.1.50
      BLENDER_PORT: 9876
      # (Optionnel) URL publique pour le manifeste MCP
      MCP_PUBLIC_URL: https://mcp.mondomaine.tld
      # (Optionnel) Protection HTTP
      MCP_BASIC_AUTH_USER: blender
      MCP_BASIC_AUTH_PASSWORD: change-me
      # Pour un token Bearer, utilisez MCP_BEARER_TOKEN à la place
    command:
      [
        "bash",
        "-lc",
        "blender-mcp --transport http --host 0.0.0.0 --port 3000"
      ]
    networks:
      - mcp_net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://127.0.0.1:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s

networks:
  mcp_net:
    driver: bridge
```

> Vérifiez la disponibilité des options `--transport/--host/--port` via `blender-mcp --help`, certaines versions de `mcp` utilisant des noms d’arguments légèrement différents.

### `deploy/Dockerfile`

```dockerfile
FROM python:3.11-slim AS base

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Dépendances systèmes minimales
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl build-essential libssl-dev && \
    rm -rf /var/lib/apt/lists/*

COPY pyproject.toml uv.lock README.md ./
COPY src ./src
COPY main.py addon.py ./ 

RUN pip install --upgrade pip setuptools wheel && \
    pip install "mcp[cli]>=1.3.0" && \
    pip install .

EXPOSE 3000

ENTRYPOINT ["python", "-m", "blender_mcp.server"]
```

Construisez et démarrez le service (depuis `deploy/`) :

```bash
docker compose -f docker-compose.mistral.yml up -d --build
```

Le serveur MCP écoutera sur `http://0.0.0.0:3000`. Le reverse proxy Synology terminera TLS et ajoutera l’authentification.

## 5. Sécurisation applicative

1. **Authentification via reverse proxy** : utilisez Basic Auth ou un fournisseur OAuth2 sur Synology pour protéger l’URL publique.
2. **Limiter les origines** : appliquez des règles IP ou des ACL directement dans le reverse proxy ou via `MCP_ALLOWED_ORIGINS`.
3. **Observabilité** : activez la journalisation (niveau INFO) et expédiez les logs vers une stack (Grafana Loki, Synology Log Center).
4. **Rotation des mots de passe** : changez régulièrement les identifiants Basic Auth et stockez-les dans un coffre (Synology Password Manager, Vaultwarden).

## 6. Configuration Mistral AI

### 6.1 Le Chat (interface web)

1. Dans Le Chat, ouvrez *Paramètres* → *Tools & MCP* (Beta) → *Add MCP Server*.
2. Renseignez :
   - **Name** : `Blender`
   - **Endpoint** : `https://mcp.mondomaine.tld/`
   - **Headers supplémentaires** : laissez vide si l’authentification est gérée par le reverse proxy (Le Chat demandera les identifiants Basic Auth si nécessaire).
3. Testez la connexion : les outils Blender devraient apparaître dans la liste.

### 6.2 API / SDK

Lors de l’appel à l’API Mistral, ajoutez la section `mcp_servers` dans la création de session ou d’agent :

```json
{
  "mcp_servers": [
    {
      "name": "blender",
      "url": "https://mcp.mondomaine.tld/"
    }
  ]
}
```

## 7. Intégration n8n

1. Créez un *HTTP Request* node pointant vers `https://mcp.mondomaine.tld/`.
2. Renseignez l’authentification Basic Auth si elle est exigée par le reverse proxy.
3. Construisez des flux pour appeler les routes HTTP exposées par MCP (selon la version, typiquement `POST /invoke/tool` avec le nom de l’outil dans le corps).
4. Optionnel : encapsulez les appels dans un *Function* node pour gérer les payloads MCP.

## 8. Tests et exploitation

1. Vérifiez la santé via `curl https://mcp.mondomaine.tld/health` (ou ajoutez `-u user:password` / `-H "Authorization: Bearer …"` selon l’auth choisie).
2. Contrôlez que la découverte fonctionne : `curl https://mcp.mondomaine.tld/.well-known/mcp.json`.
3. Sur le NAS, surveillez `docker logs blender-mcp`.
4. Dans Blender, assurez-vous que l’addon indique « Connected to server ».
5. Script de test rapide (depuis le conteneur) :

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"tool":"get_scene_info","arguments":{}}' \
  http://127.0.0.1:3000/invoke/tool
# Ajustez l’URL selon les routes exposées par la version de FastMCP (`blender-mcp --help` affiche les chemins disponibles).
```

## 9. Bonnes pratiques supplémentaires

- **Segmentation réseau** : placez le conteneur dans un VLAN isolé, autorisez uniquement la communication vers le Mac.
- **Sauvegardes** : versionnez votre configuration (`docker-compose`, certificats) et sauvegardez le dossier `deploy/`.
- **Mises à jour** : appliquez régulièrement les mises à jour de Blender, du NAS, de Docker et recréez l’image.
- **Audit** : testez régulièrement l’exposition via `nmap`/`ssllabs`, surveillez les journaux d’accès et activez des alertes.

En suivant ces étapes, vous obtenez un déploiement sécurisé du serveur Blender-MCP, accessible depuis Internet pour Mistral AI et n8n tout en protégeant votre infrastructure domestique.
