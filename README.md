# powersync-coolify

This repository is a **local development environment** for testing a **custom Coolify one-click service** template. It provisions an Ubuntu VM with Docker and [Coolify](https://coolify.io), then injects the **PowerSync** compose template, logo, and service registry entry into the running Coolify container—similar to what you would contribute upstream (see [Add a new service template to Coolify](https://coolify.io/docs/get-started/contribute/service) and [Services introduction](https://coolify.io/docs/services/introduction)).

## Prerequisites (macOS)

### Vagrant (Homebrew)

```bash
brew install vagrant
```

### VirtualBox

Install VirtualBox from the [official VirtualBox download page](https://www.virtualbox.org/wiki/Downloads) (not via Homebrew unless you prefer the cask; the Oracle installer is the common path).

Vagrant is configured to use the **VirtualBox** provider for this project.

## How to use this repo

1. Install Vagrant and VirtualBox (above).
2. From the repository root, bring the VM up (first run downloads the box and runs provisioning):

   ```bash
   vagrant up
   ```

3. Open Coolify in your browser (see [Access Coolify](#access-coolify)).
4. After changing files under `services/`, re-run provisioning so Coolify picks up updates:

   ```bash
   vagrant provision
   ```
5. If necessary (for example inspecting containers) you can always SSH into the VM using
   
   ```bash
   vagrant ssh
   ```

## Vagrant commands

| Command | Purpose |
|--------|---------|
| `vagrant up` | Create/start the VM and run provisioning. |
| `vagrant provision` | Run provisioners again without a full rebuild (useful after editing `services/*` or the `Vagrantfile`). |
| `vagrant ssh` | Open an SSH session into the VM as the default user. |
| `vagrant destroy` | Stop and delete the VM (data inside the VM is removed unless you use external disks). |
| `vagrant reload` | Reboot the VM (e.g. after changing forwarded ports or VirtualBox settings). |


### VirtualBox GUI login

In `vagrantfile`, `vb.gui` is `false` by default. Set it to `true` in the `virtualbox` provider block if you want to open the VM console:

- **Username:** `vagrant`
- **Password:** `vagrant`

## Repository structure

```text
.
├── vagrantfile
│   VM definition: Ubuntu 24.04, port forwards, synced `./services` →
│   `/home/vagrant/services`, Docker + Coolify install script, and a provisioner
│   that copies PowerSync assets into the Coolify container and patches
│   `service-templates.json`.
├── services/
│   ├── powersync.yaml
│   │   Coolify-style Docker Compose template for PowerSync (metadata comments +
│   │   `services.powersync` definition).
│   ├── powersync.svg
│   │   Service logo; copied into Coolify's `public/svgs/` path expected by the
│   │   template.
```

## Access Coolify

Port forwarding from the `vagrantfile`:

- **Coolify dashboard:** [http://localhost:8000](http://localhost:8000) (guest `8000` → host `8000`).
- **HTTP on the VM (often proxy / apps on port 80):** [http://localhost:8080](http://localhost:8080) (guest `80` → host `8080`).
- **HTTPS:** host `8443` → guest `443`.
- **Extra range:** guest ports `6000`–`6060` are also forwarded to the same ports on the host.

Complete the Coolify first-time setup in the UI as prompted.

## Try the PowerSync one-click service

1. In Coolify, **create a new project** (or open an existing one).
2. **Add a new resource** → **Service** (one-click / compose-based service).
3. **Search for `powersync`** and select the PowerSync template.
4. Configure required environment variables (see `services/powersync.yaml`), especially:
   - `PS_DATA_SOURCE_URI`
   - `PS_SYNC_BUCKET_URI`
   - `PS_BACKEND_JWKS_URI`

Deploy and verify logs and health checks from the Coolify UI.

---

## Example: PowerSync + Supabase

This walks through deploying Supabase and PowerSync together in Coolify using one-click templates.

### 1. Deploy Supabase

1. Add a new resource → **Service** → search for **Supabase** and select it.
2. Before deploying, go to the Supabase service **Settings** and enable **"Connect To Predefined Network"**. This is required so PowerSync can reach the Supabase database.
3. Deploy and wait for all containers to become healthy.

### 2. Access Supabase Studio from the host (optional)

Coolify assigns a domain using the VM's external IP, which isn't reachable from the host machine. To fix this:

1. Go to the Supabase service → click **supabase-kong** → find the domain/URL field.
2. Replace the external IP with the VM's private IP (`192.168.56.10`). For example:
   - **Before:** `http://supabasekong-<service-id>.161.142.146.132.sslip.io`
   - **After:** `http://supabasekong-<service-id>.192.168.56.10.sslip.io`
3. Redeploy the Supabase service and visit the updated URL.

### 3. Gather Supabase connection details

You need two things:

- **Service ID** — visible in the Supabase Kong URL. If the URL is `http://supabasekong-ae2yug3bigzjp3rcm7svy4ou.192.168.56.10.sslip.io`, the service ID is `ae2yug3bigzjp3rcm7svy4ou`.
- **Postgres password** — found in the Supabase service environment variables on the General tab (`SERVICE_PASSWORD_POSTGRES`).

The database connection URL follows this pattern:

```
postgresql://postgres:<PASSWORD>@supabase-db-<SERVICE_ID>:5432/postgres
```

### 4. Deploy PowerSync

1. Add a new resource → **Service** → search for **PowerSync**.
2. Set these environment variables:

   | Variable | Value |
   |----------|-------|
   | `PS_DATA_SOURCE_URI` | `postgresql://postgres:<password>@supabase-db-<service-id>:5432/postgres` |
   | `PS_SYNC_BUCKET_URI` | Same as above (can use the same database) |
   | `PS_BACKEND_JWKS_URI` | `http://supabasekong-<service-id>.192.168.56.10.sslip.io/auth/v1/.well-known/jwks.json` |

3. Deploy PowerSync.

### 5. Configure the database for replication

PowerSync requires logical replication. Follow the [PowerSync database setup guide](https://docs.powersync.com/configuration/source-db/setup#1-ensure-logical-replication-is-enabled), then create the publication:

```sql
CREATE PUBLICATION powersync FOR ALL TABLES;
```

### 6. Update sync config

The template ships with placeholder sync streams referencing `mytable`. Replace them with your actual tables:

1. In Coolify, go to the PowerSync service → **Storages** → **Files**.
2. Edit **sync-config.yaml** with your sync stream definitions.
3. Save and redeploy.

See the [sync streams documentation](https://docs.powersync.com/sync/streams/overview) for details.

---

## Template reference

### Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `PS_DATA_SOURCE_URI` | Yes | Source database connection string (`postgresql://user:pass@host:5432/db`) |
| `PS_SYNC_BUCKET_URI` | Yes | Sync bucket storage connection string (can be the same database) |
| `PS_BACKEND_JWKS_URI` | Yes | JWKS endpoint for verifying client JWTs |
| `PS_DATA_SOURCE_TYPE` | Yes | Database type (default: `postgresql`) |
| `PS_SYNC_BUCKET_TYPE` | Yes | Storage type (default: `postgresql`) |

`PS_HS256_KEY` and `PS_API_TOKEN` are auto-generated by Coolify.

### Config files

Both files are editable in the Coolify UI under **Storages → Files**:

- **powersync.yaml** — main service configuration (ports, connections, auth).
- **sync-config.yaml** — sync streams defining which data is synced to each client.

### Cross-service networking

When connecting PowerSync to another Coolify service, enable **"Connect To Predefined Network"** in the Settings of the other service. The PowerSync template already joins this network. Use the full container name (e.g. `supabase-db-<service-id>`) as the hostname in connection strings.

---

## Further reading

- [Services — what they are in Coolify](https://coolify.io/docs/services/introduction)
- [Add a new service template to Coolify](https://coolify.io/docs/get-started/contribute/service)
- [Docker Compose — Coolify's magic environment variables](https://coolify.io/docs/knowledge-base/docker/compose#coolify-s-magic-environment-variables)
- [PowerSync self-hosting documentation](https://docs.powersync.com)
- [PowerSync sync streams overview](https://docs.powersync.com/sync/streams/overview)
- [PowerSync database setup guide](https://docs.powersync.com/configuration/source-db/setup)