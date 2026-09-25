# Bootc Cosign Guide

Manual cosign-based image signing and verification for bootc / Universal Blue
images, replacing the BlueBuild `cosign` module with an explicit file-based
setup you fully control.

Covers: key generation, public key deployment, `policy.json`, registries.d
config, tmpfiles.d boot sync, GitHub Actions secret, and rebase procedure.

Tested against:
- **Sirius-OS** (Bazzite base)
- **Wolf-OS** (Fedora Silverblue base)

---

## Why the Manual Method

The BlueBuild `cosign` module hides four moving parts: the private key
(GitHub Actions secret), the public key, the policy, and the registry
attachment config. With the module you don't know what the secret
actually contains — you can't inspect it, rotate it, or verify it matches
the public key baked into the image. This guide lays all four out
explicitly so you can:

- Audit exactly what's deployed, where, and what's kept secret
- Regenerate the key pair yourself
- Delete the repository and create a new one with the same name
- Swap keys or registries without waiting on module updates
- Understand why `/etc` shadows `/usr/etc` and how to work with it
- Rotate keys on your own schedule with full visibility

---

## File Structure

Files destined for the image live under `files/system/` in the repo and are
copied verbatim into the image root.

```text
files/
└── system/
    └── usr/
        ├── etc/
        │   ├── containers/
        │   │   ├── policy.json
        │   │   └── registries.d/
        │   │       └── sirius-os.yaml
        │   └── pki/
        │       └── containers/
        │           └── sirius-os.pub
        └── lib/
            └── tmpfiles.d/
                └── cosign-policy.conf
```
## 1. Remove the Cosign Module From the Recipe

Before adding the manual file-based setup, remove the BlueBuild `cosign`
module from your recipe. Leaving it in place while adding manual files
creates a conflict — the module writes to the same paths your `files`
entry is now responsible for, and whichever runs last wins. In practice
this produces confusing signature failures that look like a policy
problem but are actually a race between the module and your files.

### Find the module

Search your recipe for a `cosign` entry:

```bash
grep -n -A 10 'type: cosign' recipes/recipe.yml
```

It will look something like:

```yaml
  - type: cosign
    key: ...
    registry: ghcr.io/jonathonp3
```

### Remove it

Delete the entire `- type: cosign` block and its indented lines. The
manual `files` entry (added later in this guide) takes over its job.

### Also clean up module leftovers

The BlueBuild cosign module may have created files under your repo that
are now redundant:

```bash
# Check for a cosign config directory
ls -la files/cosign/ 2>/dev/null

# Check for a module-generated policy
find . -name 'policy.json' -not -path './files/system/*'
find . -name 'cosign.pub' -not -path './files/system/*'
```

If you find a generated `cosign.pub` outside `files/system/usr/etc/pki/containers/`,
it's a leftover. The guide's Section 2 will regenerate the key and place
it in the correct location, so any stale copy can be deleted.

### Verify the module is gone

```bash
grep -rn 'type: cosign' recipes/
```

Should return nothing.

### Commit

```
Remove cosign module in favor of manual file-based signing

The cosign module writes to the same paths as the manual files entry
added in the following commits, causing a race where either could win.
Removing it now so the manual setup is the single source of truth for
the key, policy, and registries.d config.
```

---

## 2. Generate the Key Pair

Run cosign in a disposable distrobox container so it doesn't pollute your host.

```bash
distrobox create --name cosign --image fedora:latest
distrobox enter cosign

mkdir -p ~/sirius-os-cosign
cd ~/sirius-os-cosign
curl -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o cosign
chmod +x cosign
./cosign generate-key-pair
```

Press **Enter** at both password prompts. A password is not supported by the
GitHub Actions signing flow used here.

This produces:
- `cosign.key` — private key (goes into GitHub Secrets, never committed)
- `cosign.pub` — public key (committed into the image)

---

## 3. Deploy the Public Key

```bash
mkdir -p ~/sirius-os/files/system/usr/etc/pki/containers
cp ~/sirius-os-cosign/cosign.pub \
   ~/sirius-os/files/system/usr/etc/pki/containers/sirius-os.pub

# Verify the copy is identical
diff ~/sirius-os-cosign/cosign.pub \
     ~/sirius-os/files/system/usr/etc/pki/containers/sirius-os.pub \
  && echo "match"
```

---

## 4. Create the Policy

`files/system/usr/etc/containers/policy.json`

```json
{
  "default": [
    { "type": "reject" }
  ],
  "transports": {
    "docker": {
      "ghcr.io/jonathonp3/sirius-os": [
        {
          "type": "sigstoreSigned",
          "keyPath": "/etc/pki/containers/sirius-os.pub",
          "signedIdentity": { "type": "matchRepository" }
        }
      ],
      "": [
        { "type": "insecureAcceptAnything" }
      ]
    },
    "docker-daemon":      { "": [{ "type": "insecureAcceptAnything" }] },
    "atomic":             { "": [{ "type": "insecureAcceptAnything" }] },
    "containers-storage": { "": [{ "type": "insecureAcceptAnything" }] },
    "dir":                { "": [{ "type": "insecureAcceptAnything" }] },
    "oci":                { "": [{ "type": "insecureAcceptAnything" }] },
    "oci-archive":        { "": [{ "type": "insecureAcceptAnything" }] },
    "docker-archive":     { "": [{ "type": "insecureAcceptAnything" }] },
    "tarball":            { "": [{ "type": "insecureAcceptAnything" }] }
  }
}
```

> **Note on the `""` catch-all:** Under `docker`, the empty-string scope with
> `insecureAcceptAnything` is intentional and matches the upstream
> ublue/secureblue pattern. It permits pulling base images from other
> registries (quay.io, docker.io) during builds and toolbox use, while your
> specific repo is still strictly verified.

---

### For Wolf-OS (Silverblue base)

Use the same structure but change the repo scope and key path to `wolf-os`,
and trim the transport list to just `docker`, `docker-daemon`, and
`containers-storage` if you prefer a minimal policy:

```json
{
  "default": [ { "type": "reject" } ],
  "transports": {
    "docker": {
      "ghcr.io/jonathonp3/wolf-os": [
        {
          "type": "sigstoreSigned",
          "keyPath": "/etc/pki/containers/wolf-os.pub",
          "signedIdentity": { "type": "matchRepository" }
        }
      ],
      "": [ { "type": "insecureAcceptAnything" } ]
    },
    "docker-daemon":      { "": [{ "type": "insecureAcceptAnything" }] },
    "containers-storage": { "": [{ "type": "insecureAcceptAnything" }] }
  }
}
```

## Policy Comparison: Strict vs Permissive

| Policy | `default` | Repo scope | Purpose |
|---|---|---|---|
| Strict | `reject` | `sigstoreSigned` | Verify known registries, reject unknown |
| Permissive | `insecureAcceptAnything` | *(absent)* | Accept everything, verification disabled |

Use the strict policy. The permissive form is only for testing or for hosts
where signature enforcement is deliberately disabled.

---

## 5. Create the Registry Config

`files/system/usr/etc/containers/registries.d/sirius-os.yaml`

```yaml
docker:
  ghcr.io/jonathonp3/sirius-os:
    use-sigstore-attachments: true
```

This tells containers tooling to look for the signature as a sigstore
attachment (the `.sig` tag) in the registry rather than as an OCI referrer.

---

## 6. Create the Boot Sync Rule

`files/system/usr/lib/tmpfiles.d/cosign-policy.conf`

```ini
# SECURITY COPIES: replace the host policy with the image policy at boot
r /etc/containers/policy.json - - - -
C /etc/containers/policy.json - - - - /usr/etc/containers/policy.json

# Ensure the key folder exists and copy the key
d /etc/pki/containers 0755 root root -
C /etc/pki/containers/sirius-os.pub - - - - /usr/etc/pki/containers/sirius-os.pub
```

`/etc` is mutable and takes precedence over `/usr/etc` on bootc systems. A
stale or permissive host copy of `policy.json` can silently shadow the
image's strict policy. This rule removes any such copy and reinstates the
image's authoritative policy and key on every boot.

---

## 7. Recipe Snippet

```yaml
  - type: files
    files:
      # Copies files/system/* into the image root (/), preserving structure:
      #   /usr/etc/containers/policy.json
      #   /usr/etc/pki/containers/sirius-os.pub
      #   /usr/etc/containers/registries.d/sirius-os.yaml
      #   /usr/lib/tmpfiles.d/cosign-policy.conf
      - source: system
        destination: /
```

---

## 8. GitHub Actions Secret

Add the **private** key as a repository secret:

1. Repo → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret**
3. Name: `SIGNING_SECRET`
4. Value: paste the full contents of `cosign.key`, including the
   `-----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----` and
   `-----END ENCRYPTED COSIGN PRIVATE KEY-----` lines
5. Save

Do **not** commit `cosign.key`. Add it to `.gitignore`:

```gitignore
cosign.key
```

---

## 9. Verify a Signed Image

From any machine with cosign installed:

```bash
cosign verify --key cosign.pub ghcr.io/jonathonp3/sirius-os:latest
```

A successful run confirms the image was signed by your private key and has
not been tampered with.

---

## 10. Rebase Procedure (One-Time)

When moving a host from an unsigned or differently-signed image onto a
newly signed image:

```bash
# 1. Make host policy permissive (temporary escape hatch)
sudo bash -c 'cat <<EOF > /etc/containers/policy.json
{"default":[{"type":"insecureAcceptAnything"}]}
EOF'

# 2. Rebase without enforcement
sudo rpm-ostree rebase ostree-unverified-registry:ghcr.io/jonathonp3/sirius-os:latest

# 3. Reboot into the new image
sudo systemctl reboot

# 4. Verify
cat /etc/containers/policy.json | head -20
ls -l /etc/pki/containers/
rpm-ostree status
```

After reboot, the active deployment should read
`ostree-image-signed:docker://...` — that confirms verification passed.

---

## 11. GHCR Package Access After Removal and the creation of a new Repository with the same name.

GHCR packages are owned by the user account, not the repo. Access grants
are tied to the repository by name, so recreating a repo (even with the
same name) does **not** inherit the old grant.


### 🛠️ Fix

1. Package settings →
   `https://github.com/users/<user>/packages/container/<image>/settings`
    
   Example: 
   `https://github.com/users/jonathonp3/packages/container/sirius-os/settings`
    
2. **Manage Actions access**

3. Find the repo (it will already be listed) and select it to enable.
4. Change the `Role` from **Read** → **Write** or **Admin**

Once this is done, the GITHUB_TOKEN in your workflow will have the necessary permissions to write.


No rebuild required. Existing digests, signatures, and tags are preserved.

### Alternative: delete the package

Deleting the package removes the page entirely. The next CI build
recreates it and grants the owning repo **Admin** by default. This works
but:
- Breaks any host or CI pulling the old package until the rebuild finishes
- Changes the digest, invalidating existing signatures
- Is not safe if a running host is mid-`bootc upgrade`

Use only if the package is already broken and nothing depends on it.

---

## License

CC BY-SA 4.0
