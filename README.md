# Ghostty Web Terminal with OSCAR CLI

This crate deploys an OSCAR exposed service that provides a browser-based terminal powered by `ghostty-web`. The terminal runs inside the service container and includes `oscar-cli`, `tmux`, and a standard Bash shell.

The service is intended to be deployed per user. OSCAR protects the exposed HTTP route with ForwardAuth, which accepts either OIDC authentication or the OSCAR service token; the terminal does not implement a second login or validate that token itself. Deploy only on clusters configured with an authenticated HTTPRoute (Traefik) and keep the terminal's internal Kubernetes Service inaccessible to untrusted workloads, because direct in-cluster access bypasses the gateway.

The container can also preconfigure `oscar-cli` against the in-cluster OSCAR API by injecting an OIDC refresh token as a secret.

## What the container provides

- `ghostty-web` frontend with a PTY-backed WebSocket session
- `oscar-cli` preinstalled in the container image
- `tmux` for optional session management inside the terminal
- Persistent workspace backed by a mounted MinIO bucket

## Workspace persistence

The example `fdl.yml` mounts the bucket path `/ghostty-web` from `minio.default` into `/mnt/ghostty-web` inside the container. The launcher script uses `WORKSPACE_DIR` directly as the working directory and also stores shell history plus `oscar-cli` user config there.

Keep `mount.path` and `WORKSPACE_DIR` aligned if you want to bind a different bucket or prefix for each user.

## Build the image

Run the build from the crate root so the Dockerfile can copy `script.sh` and the files under `docker/`:

```bash
docker build -f docker/Dockerfile -t ghcr.io/grycap/ghostty-web:0.1.0 .
```

Push the resulting image to the registry you plan to use and update `fdl.yml` if needed.

## Deploy with OSCAR CLI

Base deployment:

```bash
oscar-cli apply fdl.yml
```

The FDL declares `environment.secrets.OSCAR_OIDC_REFRESH_TOKEN` with an empty value. If you need `oscar-cli` inside the container, provide a real refresh token via a private dotenv file (`OSCAR_OIDC_REFRESH_TOKEN=<value>`) and deploy with `oscar-cli apply fdl.yml --env-file <private-env-file>`; the CLI fills the declared secret without modifying the tracked FDL. Keep the file outside the repository with restrictive permissions, and never commit the token. Leave it empty to run the terminal without preconfigured CLI credentials. When updating an existing service, providing an empty value may replace its previously configured secret. Anyone who obtains shell access can read that credential; it is not needed for browser login.

`expose.auth_type: forward` requires OSCAR's Gateway API HTTPRoute mode. The example cannot be deployed as-is on a cluster using only the legacy Ingress routing mode.

Persistent workspace:

1. Edit `fdl.yml`
2. Set `mount.path` to the bucket or prefix you want to expose in `/mnt`
3. Apply the file again

## Access the terminal

Sign in to the OSCAR Dashboard with OIDC and open the deployed terminal from the **Terminals** view. The Dashboard bootstraps an OSCAR authentication cookie before redirecting to the terminal; OSCAR checks permissions for this specific service on each gateway request. In DNS/subdomain mode its address is:

```text
https://<service-name>.<OSCAR-INGRESS-HOST>/
```

Opening the URL directly without an existing OSCAR service-auth cookie does not start OIDC login: the gateway denies access. After Dashboard bootstrap, the gateway authorizes both the page and the WebSocket upgrade.

Alternatively, supply the **OSCAR service token** in the initial URL (DNS/subdomain mode):

```text
https://<service-name>.<OSCAR-INGRESS-HOST>/?token=<oscar-service-token>
```

OSCAR's ForwardAuth validates this token and issues a service-auth cookie. Subsequent requests to the URL **without** the query string, including static assets and the terminal WebSocket, work with that cookie. The terminal application does not inspect the token. This is distinct from `OSCAR_OIDC_REFRESH_TOKEN`, which only configures `oscar-cli` inside the container. A service token grants access by possession rather than tying it to an OIDC user: do not share it when user-specific identity is required. Prefer the Dashboard/OIDC flow where possible, since tokens in URLs can leak through browser history, access logs, and copied links; navigate to the clean URL after the initial request.

When `OSCAR_OIDC_REFRESH_TOKEN` is set, the startup script generates `~/.oscar-cli/config.yaml` with a default cluster that points to the in-cluster OSCAR service endpoint:

```yaml
oscar:
  local-cluster:
    endpoint: http://oscar.oscar.svc.cluster.local:8080
    oidc_refresh_token: <refresh-token>
    ssl_verify: false
default: local-cluster
```

## Notes

- This crate assumes one deployed instance per user.
- The service is stateful from the user's point of view if a bucket is mounted, even though the exposed service itself runs as a single pod.
- If your OSCAR cluster expects `port` instead of `api_port` in the `expose` block, replace that key accordingly.
- The application does not authenticate direct traffic to its HTTP port. Do not expose that port via NodePort or an unauthenticated route, and restrict direct pod/Service access with network policy where the cluster permits it.
- The generated `oscar-cli` config is written to `~/.oscar-cli/config.yaml` inside the container and uses mode `0600`.
- If you mount `/mnt`, the workspace can persist independently from the in-container credentials. Review whether you want the generated CLI config to persist together with that workspace.
