---
name: jitsi-identity-admin-plane
description: "Operate the meet CA, mTLS console and admin plane."
version: 1.0.0
author: chrismok, Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [jitsi, pki, step-ca, mtls, nginx, admin-plane, meet.dobbscloud.com]
---

# Jitsi identity, CA and admin plane (dobbscloudeast)

## When to Use

Use when touching anything about *who may change the Jitsi site* on
dobbscloudeast: the meet.dobbscloud.com issuing CA, admin certificates, the mTLS
console front door, banishment, or applying settings to `config.js`. For the
user-facing policy (login to start, 3-room cap) and the SIP voice path see
`jitsi-operations` and `hermes-integration-sip-jitsi`.

Host: dobbscloudeast, tailnet 100.83.185.118, Ubuntu 22.04. Jitsi stack lives in
`~/jitsi-migrate-11248`; identity stack in `~/jitsi-identity`.

## Component map

| Piece | Container | Reachability |
|---|---|---|
| Identity console (Flask) | `jitsi-identity-control` | NO host port. `jitsi-identity` network only, uid 10010, `cap_drop: ALL`, `no-new-privileges` |
| mTLS front door (nginx) | `identity-proxy` | `127.0.0.1:8443`; `tailscale serve` TCP 8444 → 8443 |
| Meet issuing CA | `meet-step-ca` (smallstep/step-ca 0.30.2) | `ca.meet.dobbscloud.com:9000`, alias on `jitsi-identity` only |
| Old CA (rollback) | `jitsi-identity-ca` | Stopped. `sudo docker start jitsi-identity-ca` to roll back |
| Host applier | — | `/usr/local/sbin/jitsi-admin-apply`, root, systemd path unit |

Networks: `jitsi-identity` is created `--internal` (no egress); the proxy alone
sits also on `jitsi-identity-egress` so it can fetch the root CRL from zeppo.
The console must never be on the Jitsi `meet.jitsi` network and must never have a
host port — that was the pre-2026-11-22 design and it exposed an unauthenticated
CA and private keys.

## PKI chain

    fsck Test Root CA (P-384, 10y, zeppo SoftHSM token fsck-test-root)
      └─ meet.dobbscloud.com Signer CA (name-constrained, root-only key on
         dobbscloudeast /root/pki/meet-signer/, NOT yet loaded into step-ca)
           └─ meet-step-ca (stock step-ca, data /srv/meet-step-ca, uid 1000)
                └─ admin client certs CN=<user>@meet.dobbscloud.com

Admin console policy (step-ca): DNS `meet.dobbscloud.com` + `*.meet.dobbscloud.com`,
email `@meet.dobbscloud.com`; template forces email-only SANs + clientAuth + CDP
`http://crl.test.fsck.lan/meet.crl`; provisioner `admin-console` (JWK) with
password in `/srv/meet-step-ca/secrets/admin-console.pass`; 24 h leaf validity,
720 h provisioner. `ca.json` provisioner `policy` is NOT honoured by this
version — the template must fail closed.

CRLs: `crl.test.fsck.lan` is served by zeppo (`~/fsck-crl`, CoreDNS on
100.93.126.10:53). zeppo `pull-meet-crl.sh` mirrors every 15 min into
`~/fsck-crl/www/meet.crl`; the proxy's crl-sync pulls it into
`/srv/identity-proxy/crl/all.pem` every 600 s and fails closed. Set
`crl.idpURL` to the published URL or nginx/OpenSSL reject the CRL with
"different CRL scope".

## mTLS front door rules (`~/jitsi-identity/proxy/nginx.conf`)

- `ssl_verify_client on`, `ssl_verify_depth 2`, `ssl_crl /crl/all.pem`.
- Issuer allow-map matches exactly `CN=meet.dobbscloud.com Signer CA,O=fsck.app Test`.
  A re-issued signer with a different O makes every cert fail with 403 — check the
  map when certs suddenly stop working.
- CN must parse as `<x>@meet.dobbscloud.com`; the email is forwarded as
  `X-Admin-Email` and the serial as `X-Admin-Cert-Serial`.
- Bans are name-based, not serial-based, so they survive re-issue: add
  `<email> 1;` to `proxy/banned.map`, then `docker exec identity-proxy nginx -s reload`.
- Access log format is one line per admin action, with email + serial.
- Console app independently rejects anything whose peer is not the proxy IP
  (direct container access returns 403) — that check is the reason `wget` from
  inside the proxy container to `http://control:8080/` returns 403.

No client cert → 400 (TLS layer). Bad issuer, empty CN, or banned → 403.
`/api/v1/certs` is retired and returns 410 by design; CSRs go to meet-step-ca.

## Admin plane (settings without a web restart)

Chain: console → JSON request in staging → root applier → live `config.js`.

- Console runs unprivileged and can ONLY write
  `/srv/jitsi-admin/staging/web/request.json` (uid 10010). It never ships JS.
- `jitsi-admin-apply` (root) validates against a fixed typed allow-list (11 web-tier
  settings; nested `prejoinConfig`/`welcomePage`/`breakoutRooms` merged with
  `Object.assign`), renders `custom-config.js` itself, backs up the old file to
  `/srv/jitsi-admin/backups/`, moves staged files to `processed/`, and regenerates
  `config.js` inside the running web container with `docker exec` (no restart).
- Units: `jitsi-admin-apply.path` (PathExists on the staging file) + `.service`,
  and `jitsi-admin-report.timer` (5 min drift fingerprint → `status/live.json`).
- Modes: no args = apply pending; `--check FILE` = validate+render to stdout only;
  `--report` = refresh `status/live.json`.
- Revisions are append-only JSONL in the console volume
  `/data/admin-plane/revisions.jsonl`; roles in `roles.json`
  (static now — `jitsi-admin`, `pki-admin`; GLauth groups later, fail closed).

HARD RULE: never run `s6-web-config-override.sh` to apply a setting. That script
reissues the Let's Encrypt certificate via Cloudflare DNS-01. Re-render only the
`config.js` lines via `docker exec`. A web container restart is also not harmless
— web nginx proxies `/xmpp-websocket` and `/http-bind`.

Drift states reported: `unmanaged`, `in-sync`, `drifted` (fingerprints in
`status/live.json` vs live files).

## Cache gotcha that blocks every settings change

`config.js` is served `cache-control: max-age=14400, must-revalidate` and
Cloudflare edge-caches it. Origin already sends no-cache via
`nginx-custom/admin-plane-nocache.conf`, but Cloudflare overrides. Until a
Cloudflare Cache Rule bypasses `/config.js` and `/interface_config.js` (plus one
purge), an applied revision can take up to 4 h to be seen. Verify with
`curl -sI https://meet.dobbscloud.com/config.js | grep -iE 'cache-control|cf-cache'`.

## Nginx edits on stable-11248 (rootfs read-only, /run is tmpfs)

Write to BOTH `/root/.jitsi-meet-cfg/web/nginx-custom/` and the live tmpfs copy
`/run/web/config/nginx-custom/` (`docker exec -i … sh -c 'cat > …'`; `docker cp`
fails on the read-only rootfs). Then:

    sudo docker exec jitsi-migrate-11248-web-1 nginx -c /run/web/config/nginx/nginx.conf -t
    sudo docker exec jitsi-migrate-11248-web-1 nginx -c /run/web/config/nginx/nginx.conf -s reload

nginx runs as uid 1000 under s6; test before reload or you lose all signaling.

## Admin certificates (operator use)

User admin cert: zeppo `~/.pki/admin-console/chrismok-admin.p12` + `p12.pass`
(expires 2026-12-22 — 30-day policy, so expect to re-enroll monthly). Present it
with `curl --cert/--key` or import into the browser. Never re-issue by hand-editing
step-ca data; always go through the `admin-console` provisioner.

Tailscale TLS cert for the proxy's own server side lives in `/srv/identity-proxy/tls`
and is renewed by `/etc/cron.d/identity-proxy-cert` at 03:23 (tailscale cert only
re-issues near expiry). After renewal the cron reloads nginx.

## Verification

```bash
# no cert must be 400; direct container must be 403
curl -s -o /dev/null -w "%{http_code}\n" https://dobbscloudeast.tail687326.ts.net:8444/
sudo docker exec identity-proxy sh -c 'wget -qO- http://control:8080/ >/dev/null; echo rc=$?'
# CA health + CRL window
sudo docker exec meet-step-ca step ca health
openssl crl -in /srv/identity-proxy/crl/all.pem -noout -lastupdate -nextupdate
# admin plane
sudo cat /srv/jitsi-admin/status/live.json; sudo tail -5 /srv/jitsi-admin/apply.log
systemctl is-enabled jitsi-admin-apply.path jitsi-admin-report.timer
# serving stack
sudo docker ps --format '{{.Names}}\t{{.Status}}'
tailscale serve status
```

## Rollback

- Console cutover: `identity.yml.bak-proxy-20261122`, `identity.yml.bak-adminplane-20261122`.
- CA: stop `meet-step-ca`, `sudo docker start jitsi-identity-ca` (old console API comes back).
- Admin plane: `systemctl disable --now jitsi-admin-apply.path jitsi-admin-report.timer`;
  revision history in `revisions.jsonl`; last good `custom-config.js` in `/srv/jitsi-admin/backups/`.
- Bans: remove the line from `banned.map` and reload the proxy.
- CRL enforcement: never disable `ssl_crl` without also accepting that revoked admin
  certs work again; fix the CRL mirror instead (it fails closed by design).

## Gotchas

- `jitsi-identity` being `--internal` is load-bearing. Adding the console to
  `meet.jitsi` or publishing a port re-opens the unauthenticated-CA hole.
- Ubuntu's openssl + SoftHSM: the pkcs11 provider recurses/segfaults on this host.
  Root and intermediate signing goes through `~/.pki/test-root/ceremony.py`
  (`cryptography` builds the TBS, `pkcs11-tool` signs; keys never leave the token).
- `step ca health` inside the container needs the CA to be up; a 502 here while
  containers look healthy usually means the container was started before the
  token/DB was readable (uid 1000 owns `/srv/meet-step-ca`).
- The console's `/api/v1/whoami` is the fastest way to confirm the proxy is passing
  `X-Admin-Email` — with a client cert it echoes the email, without one you never
  reach the app.
