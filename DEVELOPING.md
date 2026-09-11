# Developing NiFi Rocks

This document describes how to build, test, and validate the Apache NiFi Rocks packaged in this repository.

---

## Installing dependencies

Building the rock needs Rockcraft and an LXD backend:

```bash
sudo snap install rockcraft --classic
sudo snap install lxd && sudo lxd init --auto
sudo adduser "$USER" lxd && newgrp lxd
```

Running the smoke tests additionally needs `just`, `yq`, `jq`, `docker`,
`goss`/`kgoss`, and a Kubernetes cluster. `just test` stands up its own docker
registry to hand the image to `kgoss`, so no separate registry addon is needed —
but `kgoss` still runs the rock as a pod, so a working cluster and kubeconfig
are required.

```bash
sudo snap install just --classic     # just is classic confinement; yq is not
sudo snap install yq
sudo apt-get install -y jq docker.io

goss_base_url="https://github.com/goss-org/goss/releases/latest/download"
sudo curl -L "${goss_base_url}/goss-linux-$(dpkg --print-architecture)" -o /usr/local/bin/goss
sudo curl -L "${goss_base_url}/kgoss" -o /usr/local/bin/kgoss
sudo chmod +rx /usr/local/bin/goss /usr/local/bin/kgoss
```

Any cluster `kgoss` can reach works; the steps below use microk8s. `microk8s
config` cannot be redirected straight into `~/.kube/config` (the confined snap
fails with `Bad file descriptor`), so pipe it through `tee`, otherwise `kubectl`
silently falls back to `localhost:8080`:

```bash
sudo snap install microk8s --classic
sudo snap install kubectl --classic
sudo usermod -a -G microk8s "$USER"
sudo chown -R "$USER" ~/.kube
newgrp microk8s                        # or log out and back in

microk8s status --wait-ready
microk8s config | tee ~/.kube/config > /dev/null
chmod 600 ~/.kube/config

kubectl get nodes                      # must succeed before `just test`
```

---

## Repository Structure

```
.
├── <major.minor>/
│   ├── rockcraft.yaml      # the rock definition
│   ├── nars.keep           # NAR allowlist applied at build time
│   ├── goss.yaml           # smoke tests run against the built image
│   ├── goss_wait.yaml      # readiness gate for the smoke tests
│   └── scripts/
│       └── start.sh        # Pebble entrypoint wrapper
├── justfile
├── DEVELOPING.md
├── LICENSE
└── README.md
```

Each version directory contains everything needed to build that NiFi release.

---

## Building the Rock

To build the NiFi Rock, go to the directory that corresponds to your desired version. For example:

```bash
cd 2.10
rockcraft pack
```

This produces a `.rock` artifact (`nifi_2.10.0_amd64.rock`) in the same directory.
Or via the justfile, from the repository root:

```bash
just pack 2.10
just clean 2.10
```

---

## Testing the Rock

`just test <version>` packs the rock, pushes it to a throwaway docker registry
it starts on `localhost:5000`, and runs the `goss.yaml` smoke tests in a pod
started from that image — pulled by digest, so the container under test is
exactly the image just pushed. The registry is torn down on exit.

`goss_wait.yaml` gates the run until NiFi answers on its API. 
`goss.yaml` asserts on things NiFi writes during startup
(`flow.json.gz`, the generated sensitive key, an error-free app log). Expect the first wait attempt to fail and the second to pass a few seconds later; that is the gate working.

```bash
just test 2.10
```

Use `just run 2.10` instead to drop into `kgoss edit`, an interactive session
against the same container - useful when adding or debugging assertions.


> The port check uses `tcp6:8080`, not `tcp:8080`. NiFi's Jetty binds a
> dual-stack IPv6 wildcard socket, so the listener appears only in
> `/proc/net/tcp6`; an IPv4-only check never matches even though the port is
> serving.

---

## Publishing the Rock

`.github/workflows/publish.yaml` publishes the rock to
`ghcr.io/canonical/nifi-rocks/nifi` whenever a change to a version directory lands on `main`. Only the version directories touched by the push are rebuilt, and changes to the goss files alone do not trigger a publish.

Each build is pushed under two tags: the major.minor line (`2.10`), which floats to the newest build and is what the charm's `upstream-source` points at, and the full version (`2.10.0`). The digest is printed in the run's job summary for pinning.

To publish without a change, for example after a failed run, trigger the workflow manually from the Actions tab, optionally naming a single version directory:

```bash
gh workflow run publish.yaml -f rock-dir=2.10
```

This is the edge image the charm consumes. Governed releases to Docker Hub go through [OCI Factory](https://github.com/canonical/oci-factory) and are separate.

---

## Artifact sourcing and verification

The rock consumes the `-ubuntuN` artifact built from source by the SOSS pipeline and published by
[central-uploader](https://github.com/canonical/central-uploader/releases), and
prunes its NARs.

The `nifi` part pins both the artifact URL and the publisher's SHA-512 in
`source-checksum`. Rockcraft verifies the digest during the pull step, so a
swapped, truncated, or corrupted artifact **fails the build** before anything is
unpacked.


---

## NAR pruning

The NiFi distribution ships 119 NARs in `lib/`. Pruning is controlled by the
`NAR_ALLOWLIST` build environment variable in `rockcraft.yaml`:

| `NAR_ALLOWLIST` | Behaviour |
| :---- | :---- |
| `nars.keep` (default) | Keep only the NARs listed in that file (23) |
| `""` | Keep every NAR the distribution ships; logged at build time |
| names a missing file | Build fails with an explicit error |

The behaviour is driven by the variable rather than by whether a file happens to
exist, so both modes are visible in `rockcraft.yaml`

`nars.keep` is an **allowlist**: anything not listed is deleted, and anything
listed that is *not* found in the distribution fails the build rather than shrinking the image.

When bumping the version, re-check that each kept NAR's `Nar-Dependency-Id`
still resolves inside the keep-set; NiFi's classloader logs unsatisfied NAR
dependencies on startup.

---

## Versioning

Each NiFi version maintains its own isolated build definition under `<major.minor>/`. Future versions follow the same structure.

---

## License

This project is licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
