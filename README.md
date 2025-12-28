# k8s-homelab-blueprint

Portable, secret-free blueprint of a homelab Kubernetes stack. It captures networking, storage, and workloads so you can share the architecture publicly or fork it as a starting point.

## High-level architecture
- **Cluster:** k3s on bare metal.
- **Networking:** MetalLB (L2) handing out LAN IPs for game/stream services; Traefik/Ingress for HTTP(S) apps; direct LoadBalancer for UDP-heavy services (e.g., CS2).
- **Storage:** RWX NFS export mounted by all nodes for stateful apps and game content.
- **Domains:** `*.example.com` (replace with your zone). HTTP apps sit behind Ingress with TLS; CS2 uses a direct LB IP.

## What’s included (manifests/)
- `namespaces.yaml` — logical separation for games, media, devops, monitoring, comms, registry.
- `metallb-pool.example.yaml` — example L2 pool; set your CIDR and interface.
- `storage-nfs.example.yaml` — shared RWX PV/PVC template; set your NFS server/path.
- `cs2-server.yaml` — VAC-friendly CS2 dedicated server with LoadBalancer + NFS mount for game files.
- `gitlab.yaml` — lightweight GitLab skeleton with external Postgres/Redis endpoints to fill in.
- `jellyfin.yaml` — media server with RWX media library claim.
- `n8n.yaml` — automation/orchestration app.
- `uptime-kuma.yaml` — monitoring/health checks.
- `registry.yaml` — container registry + ingress.
- `mail-server.yaml` — mail stack stub (SMTP/IMAP) with placeholders for creds and certs.

## Networking flow
- **MetalLB (L2):** allocates `192.168.1.70` (example) to `cs2-service` for game traffic (UDP/TCP 27015, 27020, 27005).
- **Ingress (Traefik/nginx):** fronts HTTP apps (GitLab, Jellyfin, n8n, Uptime-Kuma, registry, mail UI). TLS certs are referenced by secret names; supply your own.
- **Pods → Services → LB/Ingress:** Apps expose ClusterIP or LoadBalancer; Ingress routes hostnames to ClusterIP services on port 80/443.

## Storage
- Shared NFS export `/mnt/k3sFast/csgo` (placeholder) mounted as RWX for stateful workloads and game content.
- Each app PVC points at the shared PV; adjust sizes/paths per app.

## How to use this blueprint
1) Copy the repo folder (`k8s-homelab-blueprint`) into your project and init git.
2) Replace all `CHANGE_ME_*` placeholders:
   - Domains, TLS secret names, NFS server/path, MetalLB CIDR, Steam token for CS2, external DB endpoints for GitLab.
3) Apply in order (example):
   ```bash
   kubectl apply -f manifests/namespaces.yaml
   kubectl apply -f manifests/metallb-pool.example.yaml      # adjust first
   kubectl apply -f manifests/storage-nfs.example.yaml       # adjust first
   kubectl apply -f manifests/cs2-server.yaml
   kubectl apply -f manifests/gitlab.yaml
   kubectl apply -f manifests/jellyfin.yaml
   kubectl apply -f manifests/n8n.yaml
   kubectl apply -f manifests/uptime-kuma.yaml
   kubectl apply -f manifests/registry.yaml
   kubectl apply -f manifests/mail-server.yaml
   ```
4) Create real secrets (TLS, DB passwords, Steam GSLT) in each namespace before applying app manifests.
5) Update DNS: point `cs2.example.com` (or your host) to the MetalLB IP for CS2; point other hostnames to your ingress controller.

## Notes
- This is intentionally secret-free. All sensitive values are placeholders.
- CS2 uses a direct LoadBalancer instead of Ingress to keep UDP working and avoid proxy complications.
- RWX NFS is assumed; swap to another RWX storage class if needed.
