# IPTV on QNAP (Container Station)

Runs the `rebeliptv/iptv` container on a QNAP NAS using the `docker-compose.yml`
in this folder.

## Option A — Container Station UI (recommended)

1. Open **Container Station** on your QNAP.
2. Go to **Applications** → **Create** (on older versions: **Create Application**).
3. Give the application a name, e.g. `iptv`.
4. Paste the contents of `docker-compose.yml` into the YAML editor.
5. Click **Validate**, then **Create**. Container Station pulls the image and
   starts the stack.

## Option B — SSH

1. Enable SSH on the NAS (**Control Panel → Network & File Services → Telnet / SSH**).
2. Copy this folder to the NAS, e.g. to `/share/Container/compose/iptv/`.
3. SSH in and run:

   ```sh
   cd /share/Container/compose/iptv
   docker-compose up -d      # or: docker compose up -d (newer Container Station)
   ```

4. Check it's running:

   ```sh
   docker ps --filter name=iptv
   docker logs -f iptv
   ```

## Accessing the app

Once running, the web UI is at `http://<NAS-IP>:8080`.

> **Port 8080 conflict:** QTS's own web administration page often listens on
> port 8080. If the container fails to start with a "port is already allocated"
> error, or you land on the QNAP login page instead of the IPTV app, change the
> host port in `docker-compose.yml`, e.g.:
>
> ```yaml
> ports:
>   - "8081:8080"
> ```
>
> and access the app at `http://<NAS-IP>:8081` instead.

## Notes

- **Data persistence:** app data lives in the named Docker volume `iptv-data`,
  which survives container updates and restarts.
- **Docker socket:** the compose file mounts `/var/run/docker.sock` into the
  container. This gives the container full control over Docker on the NAS —
  effectively root access to the host. Only keep this mount if the app
  actually needs it (some apps use it for self-updating or managing sibling
  containers); otherwise remove that line.
- **Updating:** to pull a newer image later:

  ```sh
  docker-compose pull && docker-compose up -d
  ```

- **Timezone** is set to `America/Chicago`; adjust `TZ` if needed.
