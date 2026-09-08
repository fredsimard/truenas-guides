# UniFi OS Server on TrueNAS (Docker custom app)

Working procedure to have Unifi OS run on TrueNAS Scale 25 as an app, uses the image: `ghcr.io/lemker/unifi-os-server:latest`.

#### Why the Custom App in TrueNAS does not work...

The **Install Custom App** wizard cannot deploy this image: it has no field for `cgroup: host`, none for `extra_hosts`, and its Tmpfs storage type cannot set the `exec` mount option. All three are required. Deploy from a compose file instead.

### Replace placeholders

`<POOL>`, `<YOUR-APP-DATASET>`, `<UOS_HOST>` and `<TIMEZONE>` are placeholders. Substitute `<POOL>` and `<YOUR-APP-DATASET>` as you go; the other two are filled in at step 4.

---

## 1. Create the dataset

In the TrueNAS UI: **Datasets** → select your pool → **Add Dataset** → Dataset preset: Apps. Name it `unifi-os-server`, nested under whatever parent you keep applications data in. Leave the rest to their defaults otherwise.

It mounts at `/mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server`. 

> **N.B.:** Replace `<POOL>` and `<YOUR-APP-DATASET>` with your pool name and parent dataset in every command below.

### Permissions

Next make sure the following permissions are set on both this new dataset and its parent (no need to do this for the pool). The old Unifi Controller app required this to work, so it's safe to presume it still is a requirement:

| Who              | Permissions             |
| ---------------- | ----------------------- |
| User — `apps`    | *Allow \| Full Control* |
| User — `netdata` | *Allow \| Full Control* |
| Group — `docker` | *Allow \| Full Control* |

> Select ***Apply permissions recursively*** to make sure the permissions are set correctly.

## 2. Create the data directories

Connect to your TrueNAS via SSH with an administrative account, or via the Shell in the UI administration site, then issue this command:

```sh
mkdir -p /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/{persistent,var-log,data,srv,var-lib-unifi,var-lib-mongodb,etc-rabbitmq-ssl}
```

## 3. Write the compose file

Use the following command to write a file named `uos-compose.yaml` at the root of the user.

- `/sys/fs/cgroup` is the host's live kernel cgroup filesystem, **not** a directory under the app dataset. UOS runs every component as a systemd service and needs it.
- No `container_name` — the apps system assigns its own.
- `UOS_SYSTEM_IP` is only the inform address used for device adoption. It binds nothing; the GUI is reachable on whatever address the host holds.
- `<TIMEZONE>` : see [List of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). Use the value in the **TZ identifier** column, and prefer rows marked Canonical over Link. The list currently reflects release 2026b of the tz database. Or use this command in the shell of your TrueNAS: `timedatectl list-timezone`
- Modify the `11443` port as you wish, but leave the rest as is unless you know what you're doing.
- Only four ports are required to be published. Optional ones, if wanted:

| Port | Explanation |
| --- | --- |
| `5005:5005` | Used for remote debug operations and diagnostics |
| `9543:9543` | Used for UniFi Talk application communication and management |
| `6789:6789` | UniFi mobile speed test |
| `8444:8444` | Secure Portal for Hotspot |
| `28082:28082` | Device packet capture and support file downloads |
| `5671:5671` | Traffic Flow logging for UXGs adopted on L2 or L3 networks |
| `8880:8880`, `8881:8881` and `8882:8882` | Hotspot portal redirection (HTTP) |
| `5514:5514/udp` | Remote syslog capture |
| `127.0.0.1:11084` | Used for UniFi Talk TURN (Traversal Using Relays around NAT) services to help handle audio and media traffic streams for VoIP phones behind firewalls |

```sh
cat > ~/uos-compose.yaml << 'EOF'
services:
  unifi-os-server:
    image: ghcr.io/lemker/unifi-os-server:latest
    cgroup: host
    cap_add:
      - NET_RAW
      - NET_ADMIN
    extra_hosts:
      host.docker.internal: host-gateway
    tmpfs:
      - /run:exec
      - /run/lock
      - /tmp:exec
      - /var/lib/journal
      - /var/opt/unifi/tmp:size=64m
    environment:
      - TZ=<TIMEZONE>
      - UOS_SYSTEM_IP=<UOS_HOST>
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/persistent:/persistent
      - /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/var-log:/var/log
      - /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/data:/data
      - /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/srv:/srv
      - /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/var-lib-unifi:/var/lib/unifi
      - /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/var-lib-mongodb:/var/lib/mongodb
      - /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/etc-rabbitmq-ssl:/etc/rabbitmq/ssl
    ports:
      - 11443:443
      - 8080:8080
      - 3478:3478/udp
      - 10003:10003/udp
    restart: unless-stopped
EOF
```

## 4. Substitute your own values

Edit `~/uos-compose.yaml` and replace the placeholders:

| Placeholder | Replace with |
| --- | --- |
| `<POOL>` | Your pool name, so the volume paths match the dataset from step 1 |
| `<YOUR-APP-DATASET>` | The name of the dataset that contains your applications' data. |
| `<UOS_HOST>` | The hostname (recommended) or IP address of your TrueNAS host, e.g. `unifi.example.com` or `192.168.1.50` |
| `<TIMEZONE>` | Your tz database name, e.g. `America/Montreal`, `Europe/Berlin`, `UTC` |

Confirm nothing is left unsubstituted:

```sh
grep -n '<POOL>\|<YOUR-APP-DATASET>\|<UOS_HOST>\|<TIMEZONE>' ~/uos-compose.yaml
```

That should print nothing if everything has been subsituted.

## 5. Deploy through the TrueNAS apps system

> ⚠️ **Expect this first boot to be broken.** The container will come up and stay running, but Postgres, the Network application and the web API will all fail. That is normal at this stage — the image only creates its directory tree under `/data` on this first run, and until that tree exists there is nothing to correct. Step 6 fixes it. Do not skip ahead or redeploy.

Build the API payload:

```sh
jq -n --rawfile c ~/uos-compose.yaml \
  '{app_name:"unifi-os-server",custom_app:true,custom_compose_config_string:$c}' \
  > ~/uos-payload.json
```

Create the app:

```sh
midclt call -j app.create "$(cat ~/uos-payload.json)"
```

- `-j` waits on the job and shows progress. 
- Passing `-` to read stdin does **not** work — `midclt` treats it as a literal string argument and the call fails with an `AttributeError` on job lock handling.

The app then appears and is managed normally in the TrueNAS UI.

## 6. Fix directory permissions

This is the step everything hinges on.

The image `chowns` the leaf directories its services need, but bind-mounting over `/data` leaves the intermediate directories `root:root 770`. Non-root services (`postgres` uid 10100, `unifi` uid 997, `nginx`) cannot traverse into them and fail — each with a different, misleading symptom.

This has to be done in two passes, because `unifi-core` only generates `config/http/` late in its startup, and it cannot get there until the earlier failures are cleared.

### 6a. First pass

Once the container has booted and created its tree under `/data`:

```sh
cd /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/data
chmod 775 . postgresql postgresql/14 unifi unifi-core
```

`775` adds traverse and read without changing ownership.

Only the `root:root` directories need this. The others (`ucs-agent`, `uid`, `ulp-go`, `unifi-directory`, `unifi-identity-update`) are already owned by their own service uids — leave them alone. `ls -la` will show you which is which.

Restart the services that already failed:

```sh
docker exec ix-unifi-os-server-unifi-os-server-1 systemctl restart postgresql-cluster@14-main
docker exec ix-unifi-os-server-unifi-os-server-1 systemctl reset-failed unifi.service
docker exec ix-unifi-os-server-unifi-os-server-1 systemctl start unifi
```

> Replace `ix-unifi-os-server-unifi-os-server-1` with your container name if it differs. Find it with `docker ps --format '{{.Names}}' | grep unifi`.

- `reset-failed` is needed because `systemd` will have hit its start-limit ("Start request repeated too quickly") after five attempts.
- `unifi.service` is a JVM and takes a minute or two to open port 8081.

### 6b. Second pass

Wait for `unifi-core` to finish starting. It will report `activating` for several minutes before going `active`:

```sh
docker exec ix-unifi-os-server-unifi-os-server-1 systemctl is-active unifi-core
```

Once it returns `active`, `config/` will have filled with `.yaml` files, certificates and an `http/` subdirectory. Then:

```sh
cd /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/data
chmod 775 unifi-core/config unifi-core/config/http
```

That last directory holds the `uos-http.sock` unix socket that nginx proxies every `/api/` route to. Until `nginx` can traverse into it, the web UI loads but stays blank, and every API call returns a JSON 502 with nothing written to the nginx error log.

If `chmod` reports `No such file or directory` on `unifi-core/config/http`, `unifi-core` has not finished starting — wait and retry rather than skipping it.

## 7. Verify

```sh
docker exec ix-unifi-os-server-unifi-os-server-1 systemctl list-units --failed --no-pager
curl -kIs https://<UOS_HOST>:11443/api/system | head -1

## or
curl -kIs https://127.0.0.1:11443/api/system | head -1
```

`HTTP/2 200` on a console that has not been through initial setup, or `HTTP/2 401` on one where you have already created an account — both mean the API is reachable and step 6 worked. Only `HTTP/2 502` indicates the permissions are still wrong.

Empty output means nothing is listening yet. Give it more time.

Then open `https://<UOS_HOST>:11443` and you should see the setup wizard.

## 8. Changing configuration later

To change an environment variable, a port, or a volume, edit the compose file and push it back through the apps system. There is no image to rebuild — the image is pulled as-is; only your compose config changes.

Do **not** edit the rendered file under `/mnt/.ix-apps/`. It is regenerated from the app config and your changes will be overwritten.

Edit `/root/uos-compose.yaml`, then confirm the change landed:

```sh
grep UOS_SYSTEM_IP /root/uos-compose.yaml
```

Push it:

```sh
jq -n --rawfile c /root/uos-compose.yaml '{custom_compose_config_string:$c}' > /root/uos-update.json
midclt call -j app.update unifi-os-server "$(cat /root/uos-update.json)"
```

Note the argument shape differs from `app.create`: `app_name` is a separate first argument, and the compose string goes inside a second object.

This recreates the container. Your data lives on the bind mounts, so it survives, along with the permission fixes from step 6.

Expect the same startup behaviour as a fresh install: several minutes before the web UI answers, and `unifi.service` may report a failed start that resolves itself. Be patient before concluding something is broken.

After the container is recreated, re-check the directory that step 6b covered:

```sh
ls -la /mnt/<POOL>/<YOUR-APP-DATASET>/unifi-os-server/data/unifi-core/config/http/
```

The `pre-start` hook wipes and regenerates that directory's contents on every start. If the parent directories lost their `775`, reapply the 6b chmod.

---

## Symptom reference

| Symptom | Cause |
| --- | --- |
| Container restart loop, exit 255, dies in ~3s | Missing `/sys/fs/cgroup` mount and/or `cgroup: host` |
| `initdb: could not access directory "/data/postgresql/14/main/data": Permission denied` | `/data`, `/data/postgresql` or `/data/postgresql/14` not traversable by uid 10100 |
| `Error opening log file 'logs/gc.log': Permission denied`, `Could not create the Java Virtual Machine` | `/data/unifi` not traversable by uid 997 |
| Blank page, console shows 502 on `/api/*`, JSON body `{"error":{"code":502,...}}`, nothing in nginx error log | `/data/unifi-core/config/http/` not traversable by `nginx`, so the `uos-http.sock` unix socket is unreachable |
| `Path does not exist` when saving in the Custom App wizard | Unresolved — the error names no path. Not investigated further once the compose route worked. |
| Chrome `ERR_ADDRESS_UNREACHABLE` while `curl` from the same machine returns 200 | Browser-side cached state, not the server. Try a private window or another browser. |

## Diagnostic commands

`systemd` inside the container logs to the journal, not `stdout`, so `docker logs` goes quiet after the entrypoint. That is normal and not a failure.

```sh
C=ix-unifi-os-server-unifi-os-server-1

docker exec $C systemctl list-units --failed --no-pager
docker exec $C systemctl list-units --type=service --no-pager
docker exec $C journalctl -u <unit> --no-pager -n 40
docker exec $C tail -30 /data/unifi-core/logs/errors.log
docker exec $C ss -lntp
```

`unifi-core` keeps its own logs in `/data/unifi-core/logs/`; `errors.log` is the useful one. Its nginx upstreams are generated into `/data/unifi-core/config/http/upstream-*.conf` — read those to find which backend a route actually points at rather than guessing.

## Known caveat

`/var/opt/unifi/tmp` mounts `noexec` (Docker's tmpfs default; the upstream compose does not set `exec` on that entry either). The Network service logs `mount: /var/opt/unifi/tmp: permission denied` and `Could not unmount tmpfs from /var/opt/unifi/tmp`. Nothing has failed because of it in this deployment, but it is a candidate if odd Network application errors appear later.
