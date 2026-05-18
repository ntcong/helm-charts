# Helm Charts Repository - Agent Guidelines

## Image Tag Updates

When updating container image tags in any chart's `values.yaml`, you **must** also bump the chart version (patch version) in the corresponding `Chart.yaml`.

### Rules

1. **Always bump the chart version** when any image tag in `values.yaml` changes. Increment the patch version (e.g., `1.6.7` → `1.6.8`).
2. **Do not update charts** whose images did not change.
3. **Skip `auto-deploy-app`** — it uses a template placeholder image (`gitlab.example.com/group/project:stable`) and should never be updated.
4. **Verify** that the new image tag actually exists on the container registry before applying the change.

### Example

If `charts/servarr/values.yaml` has an image tag update:
- Edit `charts/servarr/values.yaml` with the new tag
- Edit `charts/servarr/Chart.yaml` and bump `version: X.Y.Z` → `version: X.Y.Z+1`

## Creating New Helm Charts (bjw-s Common Library)

All charts in this repo use the [bjw-s common library chart](https://bjw-s-labs.github.io/helm-charts/) as a dependency. Follow these conventions when creating a new chart.

### Directory Structure

Every chart must have exactly these 3 files:

```
charts/<chart-name>/
├── Chart.yaml
├── values.yaml
└── templates/
    └── common.yaml
```

### Chart.yaml

```yaml
apiVersion: v2
name: <chart-name>
description: <Short description>
type: application
version: 1.0.0
keywords:
  - <chart-name>
sources:
  - <upstream source URL>
maintainers:
  - name: ntcong
    email: ntcong.it@gmail.com
dependencies:
  - name: common
    repository: https://bjw-s-labs.github.io/helm-charts/
    version: 3.7.3
```

### templates/common.yaml

Always identical across all charts — copy verbatim:

```yaml
---
{{- include "bjw-s.common.loader.init" . }}

{{- define "app-template.hardcodedValues" -}}
# Set the nameOverride based on the release name if no override has been set
{{ if not .Values.global.nameOverride }}
global:
  nameOverride: "{{ .Release.Name }}"
{{ end }}
{{- end -}}
{{- $_ := mergeOverwrite .Values (include "app-template.hardcodedValues" . | fromYaml) -}}

{{/* Render the templates */}}
{{ include "bjw-s.common.loader.generate" . }}
```

### values.yaml Conventions

#### Service

Map each controller to a named service entry. The service name typically matches the controller name. Port name is `http` for web UIs:

```yaml
service:
  main:                              # service name (matches controller name for single-app charts)
    controller: main                 # references the controller key
    ports:
      http:
        port: 8465                   # container port
```

For multi-service charts (like servarr), create one service per controller:

```yaml
service:
  sonarr:
    controller: sonarr
    ports:
      http:
        port: 8989
  radarr:
    controller: radarr
    ports:
      http:
        port: 7878
```

For additional non-HTTP ports (e.g., BitTorrent), add a separate service:

```yaml
service:
  main:
    controller: main
    ports:
      http:
        port: 8080
  bittorrent:
    controller: main
    enabled: true
    ports:
      bittorrent:
        enabled: true
        port: 6881
        protocol: TCP
```

#### Controllers

Define one controller per application. For single-app charts, use `main` as the key:

```yaml
controllers:
  main:
    containers:
      main:
        image:
          repository: luketanti/aerofoil
          tag: 1.2.3
          pullPolicy: IfNotPresent
        env:
          TZ: UTC
```

For multi-app charts, use the application name as the controller key (e.g., `sonarr`, `radarr`).

**Important:** Always use a **specific version tag** (e.g., `1.2.3`, `v4.0.17.2953`), never `latest`. Pinning to a specific version ensures reproducible deployments and prevents unexpected breakage from upstream changes.

#### Persistence

**Local PVC (default for config/data):**

Use `storageClass: longhorn`, `retain: true`, and `advancedMounts` to scope mounts to specific controllers and containers:

```yaml
persistence:
  config:
    enabled: true
    storageClass: longhorn
    accessMode: ReadWriteOnce
    size: 1Gi
    retain: true
    advancedMounts:
      main:                          # controller name
        main:                        # container name
          - path: /app/config        # mount path inside container
```

**NFS mounts (for shared media/games storage):**

Disabled by default. Uncomment and configure the NFS server/path when deploying:

```yaml
persistence:
  games:
    enabled: false
    # type: nfs
    # server: 192.168.1.2
    # path: /volume1/games
    # advancedMounts:
    #   main:
    #     main:
    #       - path: /games
```

**Key rules:**
- Always use `advancedMounts` (not `globalMounts`) to explicitly target controller → container → path
- Always set `storageClass: longhorn` and `retain: true` for PVCs
- NFS volumes are disabled by default with commented-out config
- Config volumes are typically 1Gi; data volumes sized per application needs

#### Ingress

Disabled by default, one entry per service:

```yaml
ingress:
  main:
    enabled: false
```

For multi-app charts, one ingress entry per service:

```yaml
ingress:
  sonarr:
    enabled: false
  radarr:
    enabled: false
```

#### Environment Variables

- Always include `TZ: UTC`
- Comment out optional env vars rather than omitting them, so users can see what's available
- Use `PUID`/`PGID` for LinuxServer.io images or apps that support them