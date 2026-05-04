# Cloudflare Zero Trust

L'infrastructure s'appuie sur Cloudflare pour exposer les services de manière ultra-sécurisée, sans ouvrir le moindre port sur la Freebox.

## Principe du Tunnel Cloudflared

Un démon `cloudflared` tourne directement sur l'hôte Proxmox (géré à distance via Token). Il maintient des connexions sortantes (tunnels chiffrés persistants) vers les datacenters de Cloudflare.

```mermaid
sequenceDiagram
    participant User as 👤 Utilisateur
    participant CF as ☁️ Cloudflare Edge
    participant CF_Daemon as 🔒 Cloudflared (PVE)
    participant NPM as 🔀 NPM (10.10.10.18)
    
    User->>CF: Requête https://vault.zorko.xyz
    Note over CF: Vérification WAF &<br/>Cloudflare Access (Auth)
    CF->>CF_Daemon: Route le trafic via le tunnel actif
    CF_Daemon->>NPM: Transfère la requête locale (port 80)
    NPM->>NPM: Analyse SNI & Reverse Proxy
    NPM->>User: Renvoie la réponse du conteneur
```

## Cloudflare Access (Identity Aware Proxy)

Pour les services critiques (non publics), Cloudflare Access exige une authentification forte (SSO, OTP) **avant** même de router le paquet vers le tunnel.
- Le serveur local ne voit jamais les tentatives de bruteforce, Cloudflare absorbe tout.
- La véritable adresse IP de la maison est masquée.
