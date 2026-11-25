# Runbook: Música public (Cloudflare Tunnel + Access)

The steps I ran on Nov 7, 2026 to put Música (Navidrome) on the internet. In order, with what I should see after each one. Every command here is one I ran that day. Run them on the machine that has `kubectl`, `cloudflared` and `sops`, inside `~/Developer/homelab-brasil` unless it says otherwise.

| Thing | Value |
| --- | --- |
| Public address | `musica.projectpattie.com` |
| Tunnel name | `brasil` |
| Namespace | `cloudflared-brasil` |
| Files | `apps/base/cloudflared/` (namespace, secret, configmap, deployment, kustomization) |
| Secret name | `tunnel-credentials`, key `credentials.json` |
| Where traffic goes | `navidrome.brasil.svc.cluster.local:80` |
| Copies of cloudflared | 2 |
| Flux Kustomization | `brasil-apps` |
| Login in front | Cloudflare Access, then Navidrome's own login |

## Part 1: check the router

The tunnel connects out, so nothing should be open coming in.

```
ip route
```

I should see: a line starting with `default via`. That address is the router.

Open the router's admin page in a browser and check:

- Port forwards: none
- DMZ: off
- UPnP: off

## Part 2: make the tunnel

```
cloudflared --version
cloudflared tunnel login
```

I should see: a browser opens. Pick projectpattie.com. A `cert.pem` lands in `~/.cloudflared/`.

```
cloudflared tunnel create brasil
cloudflared tunnel list
```

I should see: `brasil` in the list next to `homelab-kubecraft`, with no connections yet.

```
cloudflared tunnel route dns brasil musica.projectpattie.com
```

I should see: a message that it added a CNAME for musica.projectpattie.com that routes to this tunnel.

```
ls ~/.cloudflared
```

I should see: `cert.pem` and one `.json` file named after the tunnel ID. That JSON is the tunnel's password. Do not paste it anywhere.

## Part 3: turn on SOPS for homelab-brasil

First check Flux already has the age key:

```
kubectl get secret sops-age -n flux-system
```

I should see: the secret listed. If it is missing, stop. SOPS has to be set up before anything else here.

Reuse the rules from the other lab:

```
cp ~/Developer/homelab-kubecraft/.sops.yaml ~/Developer/homelab-brasil/.sops.yaml
```

Then add the decryption block to `clusters/staging/apps.yaml` (provider `sops`, secret `sops-age`), and let the cluster check the file before pushing:

```
kubectl apply --dry-run=server -f clusters/staging/apps.yaml
```

Push it:

```
git add .sops.yaml clusters/staging/apps.yaml
git commit -m "feat: enable SOPS decryption for brasil-apps"
git push
```

Tell Flux to look now, then check the live object:

```
flux reconcile kustomization flux-system --with-source
kubectl get kustomization brasil-apps -n flux-system -o yaml | grep -A4 decryption
```

I should see: `decryption`, then provider `sops`, then the secret name `sops-age`. (The `force: false` line after it is just the next setting.)

It is `flux-system` in the reconcile because that is the one that reads `clusters/staging`, where `apps.yaml` lives.

## Part 4: make the encrypted secret

In this order. Do not `git add` in between.

```
kubectl create secret generic tunnel-credentials \
  -n cloudflared-brasil \
  --from-file=credentials.json=$HOME/.cloudflared/<TUNNEL-ID>.json \
  --dry-run=client -o yaml > apps/base/cloudflared/secret.yaml
```

I should see: nothing. The `>` sends the output into the file. Swap `<TUNNEL-ID>` for the real one (type the start, press Tab).

```
sops --encrypt --in-place apps/base/cloudflared/secret.yaml
cat apps/base/cloudflared/secret.yaml
```

I should see: `ENC[` where the credentials used to be. If I can still read the value, it is not encrypted. Do not commit it.

## Part 5: the other files

Copied from KubeCraft's `apps/base/cloudflared/`, then changed. For example:

```
cp ~/Developer/homelab-kubecraft/apps/base/cloudflared/deployment.yaml apps/base/cloudflared/
```

What has to be right in each file:

| File | What to check |
| --- | --- |
| namespace | name is `cloudflared-brasil`, not `cloudflared` (KubeCraft already uses that one) |
| configmap | `tunnel: brasil` (must match the tunnel name exactly, not `homelab-brasil`) |
| configmap | credentials file path is `/etc/cloudflared/creds/credentials.json` |
| configmap | one entry for `musica.projectpattie.com` going to `http://navidrome.brasil.svc.cluster.local:80`, and the 404 catch-all as the last line |
| deployment | `namespace: cloudflared-brasil` (the only line I changed) |
| deployment | `secretName: tunnel-credentials`, `replicas: 2` |

To read the real files:

```
cat apps/base/cloudflared/configmap.yaml
cat apps/base/cloudflared/deployment.yaml
```

Then Flux has to be told the folder exists. `apps/staging/kustomization.yaml` needs both lines:

```
resources:
  - ../base/fsi-scraper
  - ../base/cloudflared
```

Add the new line under the old one. Do not replace it. `brasil-apps` has `prune: true`, so if `fsi-scraper` leaves this list, Flux deletes the scraper from the cluster.

## Part 6: check, push, deploy

```
kubectl kustomize apps/staging
```

I should see: a lot of YAML and no error at the end. This catches typos and missing files before Flux does.

```
git status
git add apps/base/cloudflared apps/staging/kustomization.yaml
git commit -m "feat: add cloudflared tunnel for musica"
git push
```

`git status` first. Only the new cloudflared folder and the staging kustomization should show.

```
flux reconcile kustomization brasil-apps --with-source
kubectl get pods -n cloudflared-brasil -w
```

I should see: two cloudflared pods, both Running. Ctrl+C to stop watching.

```
cloudflared tunnel list
```

I should see: `brasil` now has connections.

## Part 7: test from outside

On my phone, Wi-Fi off, open `musica.projectpattie.com`.

I should see: the Navidrome login page (before Access is on), with https.

## Part 8: Cloudflare Access (all clicks)

In the Zero Trust dashboard (Cloudflare One), free plan:

1. Access controls, then Applications, then add an application. Pick Self-hosted.
2. Subdomain `musica`, domain `projectpattie.com`.
3. Policy 1: name `craven only`, action Allow, include my email.
4. Integrations, then Identity providers: add One-time PIN.
5. Back on the Música application, policy 2: name `demo visitors`, action Allow, include Everyone. ("Everyone" is not at the top of the list. Scroll.)

Test in a different browser: I should see the Cloudflare page asking for an email, then a code.

## Part 9: demo user in Navidrome

Log in to Navidrome as admin, open Users, add a user named `demo`. Regular user, not admin. The password does not go in this file.

## Part 10: link the tile (in `~/Developer/projectpattie`)

Three edits:

1. `src/content/config.ts`: add `href: z.string().url().optional(),`
2. `src/pages/brasil/demo.astro`: tile links use `` t.data.href ?? `/brasil/${slugOf(t)}` ``
3. `src/content/demo/05-musica.md`: add `href: "https://musica.projectpattie.com"`

```
npm run build && npm run preview
```

I should see, at `localhost:4321/brasil/demo`: the Música tile opens the Cloudflare login, and any other tile still opens its write-up page. Ctrl+C to stop.

```
git status
git add src/content/config.ts src/pages/brasil/demo.astro src/content/demo/05-musica.md
git commit -m "feat: link Música tile straight to the app"
git push
```

The site deploys through Cloudflare Pages, not GitHub Actions. Give it a couple of minutes, then check in a private tab.

## When it breaks

| What I see | First thing to check |
| --- | --- |
| Address does not load at all | `kubectl get pods -n cloudflared-brasil` (want 2 Running) |
| Pods are not Running | `kubectl logs -n cloudflared-brasil deploy/cloudflared` |
| Pods Running, still nothing | `cloudflared tunnel info brasil` (want active connections), then `tunnel:` in the configmap is exactly `brasil` |
| Flux did not apply my change | `kubectl kustomize apps/staging` for errors, then `flux reconcile kustomization brasil-apps --with-source` |
| Something else disappeared from the cluster | `cat apps/staging/kustomization.yaml`. A line got replaced instead of added. |
| Tile still opens the old page | Cloudflare Pages is still building, or the browser has the old page saved. Wait, then private tab. |

## Not done

- Finish copying the library, and decide what music belongs on a public demo
- Phone apps cannot get past Access. Browser only for now.

Done since: the demo login is on the Música tile and in Try it, and `ND_ENABLEUSEREDITING: "false"` in `navidrome.yaml` stops the demo user changing its password (Nov 24, commit `cbd8014`).
