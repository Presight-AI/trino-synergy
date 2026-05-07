# Trino Docker Image Build

Instructions for building the Trino server and packaging it into a Docker image.

## Prerequisites

- **Java 25** (Temurin or Oracle JDK — other vendors are not supported by Trino's enforcer rules)
- **Maven** (the Maven wrapper `./mvnw` is included in the repo)
- **Docker** with `buildx` enabled (for multi-architecture builds)
- **Git**
### Verify your Java version

```bash
mvn -version
```

You should see Java `25.0.1` or later, vendor `Temurin` or `Oracle` (recommended by trino).

### Setting up Java 25 (macOS)

If you don't have Temurin 25 installed:

```bash
brew install --cask temurin@25
```

Point your shell at it (add to `~/.zshrc` or `~/.bashrc`):

```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 25)
export PATH="$JAVA_HOME/bin:$PATH"
```

Reload your shell, then confirm:

```bash
source ~/.zshrc
mvn -version
```

## Build Steps

### 1. Build Trino with Maven

From the repository root:

```bash
./mvnw clean install -DskipTests
```

This compiles all modules and produces the server tarball under `core/trino-server/target/`. Tests are skipped to keep the build fast — drop `-DskipTests` if you want the full suite.

> **Note:** A full Trino build is large and may take 15–30 minutes on first run while dependencies are downloaded.

### 2. Build the Docker image

Once the Maven build succeeds, build the Docker image:

```bash
./core/docker/build.sh
```

By default this builds for `amd64`, `arm64`, and `ppc64le` and tags the image as `trino`.

## `build.sh` Options

```
Usage: ./core/docker/build.sh [-h] [-a <ARCHITECTURES>] [-r <VERSION>]
 
  -h       Display help
  -a       Build the specified comma-separated architectures
           (defaults to amd64,arm64,ppc64le)
  -p       Use the specified server package (artifact id)
           e.g. trino-server (default), trino-server-core
  -t       Image tag name (defaults to trino)
  -r       Build the specified Trino release version
           (downloads all required artifacts)
  -j       Build the Trino release with the specified JDK distribution
  -x       Skip image tests
```

## Common Examples

**Build for Apple Silicon and Intel only, skip image tests, custom tag:**

```bash
./core/docker/build.sh -a amd64,arm64 -x -t 480-synergy-trino
```

## Push to Azure Container Registry

The local build produces **two separate single-arch images** (e.g. `480-synergy-trino:480-amd64` and `480-synergy-trino:480-arm64`). To publish them as a single multi-arch image in ACR, push each arch individually and then combine them into one manifest list.

### Tag conventions

| Local image                          | ACR tag (intermediate)                                                | Final combined tag                                          |
|--------------------------------------|------------------------------------------------------------------------|-------------------------------------------------------------|
| `480-synergy-trino:480-amd64`        | `vista.azurecr.io/3rd-party/trino/trino:480-synergy-amd64`             | `vista.azurecr.io/3rd-party/trino/trino:480-synergy`        |
| `480-synergy-trino:480-arm64`        | `vista.azurecr.io/3rd-party/trino/trino:480-synergy-arm64`             | (same as above — multi-arch manifest)                        |

The arch-suffixed tags exist only as building blocks. Consumers should pull the unsuffixed tag (`:480-synergy`) — Docker resolves the correct architecture automatically.

### 1. Log in to ACR

```bash
az acr login --name vista
```

### 2. Tag the local images for ACR

```bash
docker tag 480-synergy-trino:480-amd64 vista.azurecr.io/3rd-party/trino/trino:480-synergy-amd64
docker tag 480-synergy-trino:480-arm64 vista.azurecr.io/3rd-party/trino/trino:480-synergy-arm64
```

### 3. Push both arch-specific images

```bash
docker push vista.azurecr.io/3rd-party/trino/trino:480-synergy-amd64
docker push vista.azurecr.io/3rd-party/trino/trino:480-synergy-arm64
```

### 4. Create the multi-arch manifest

Use `docker buildx imagetools create` — it handles attestation manifests correctly and works server-side in ACR. Avoid `docker manifest create`; it fails with `"is a manifest list"` if the source images already carry attestations.

```bash
docker buildx imagetools create \
  -t vista.azurecr.io/3rd-party/trino/trino:480-synergy \
  vista.azurecr.io/3rd-party/trino/trino:480-synergy-amd64 \
  vista.azurecr.io/3rd-party/trino/trino:480-synergy-arm64
```

### 5. Verify the manifest

```bash
docker buildx imagetools inspect vista.azurecr.io/3rd-party/trino/trino:480-synergy
```

You should see entries for both `linux/amd64` and `linux/arm64` under `Manifests`. You may also see two `unknown/unknown` entries — those are buildx attestation manifests (SBOM/provenance metadata) and are safe to ignore. They will not be pulled as images.

### 6. (Optional) Remove `unknown/unknown` attestation entries

If `imagetools inspect` shows `unknown/unknown` platform entries alongside the real architectures, those are buildx-generated attestation manifests (SBOM and provenance metadata). They're harmless — Docker and Kubernetes ignore them when pulling — but you can strip them out by recreating the manifest from the **image digests** rather than the tags. Referencing digests bypasses the attestation indexes entirely.

First, grab the digest for each real platform from the inspect output:

```bash
docker buildx imagetools inspect vista.azurecr.io/3rd-party/trino/trino:480-synergy
```

Look for the `linux/amd64` and `linux/arm64` entries (skip the `unknown/unknown` ones) and copy their `sha256:...` digests. Then recreate the manifest:

```bash
docker buildx imagetools create \
  -t vista.azurecr.io/3rd-party/trino/trino:480-synergy \
  vista.azurecr.io/3rd-party/trino/trino@sha256:<AMD64_DIGEST> \
  vista.azurecr.io/3rd-party/trino/trino@sha256:<ARM64_DIGEST>
```

Verify — only `linux/amd64` and `linux/arm64` should remain:

```bash
docker buildx imagetools inspect vista.azurecr.io/3rd-party/trino/trino:480-synergy
```

### 7. (Optional) Clean up arch-specific tags

Once the combined manifest is in place, the arch-suffixed tags can be removed. The underlying image blobs remain because the manifest list still references them.

```bash
az acr repository delete \
  --name vista \
  --image 3rd-party/trino/trino:480-synergy-amd64 \
  --yes
 
az acr repository delete \
  --name vista \
  --image 3rd-party/trino/trino:480-synergy-arm64 \
  --yes
```

Many teams keep the per-arch tags around for debugging — this step is optional.
### Pulling the published image

From any host:

```bash
docker pull vista.azurecr.io/3rd-party/trino/trino:480-synergy
```

Docker selects the right architecture based on the host. To confirm:

```bash
docker inspect vista.azurecr.io/3rd-party/trino/trino:480-synergy \
  --format '{{.Architecture}}'
```

## Troubleshooting

**`RequireJavaVersion` / `RequireJavaVendor` enforcer failure**
Maven is using the wrong JDK. Check `mvn -version` and reset `JAVA_HOME` to a Temurin or Oracle JDK 25.0.1+.

**`buildx` errors on multi-arch builds**
Make sure Docker buildx is set up:

```bash
docker buildx create --use --name trino-builder
docker buildx inspect --bootstrap
```

**Maven can't resolve `google-cloud-*-bom` artifacts from a private registry**
This usually means a private repository (e.g. GitHub Packages) is listed before Maven Central in `pom.xml` or `~/.m2/settings.xml`. Reorder repositories so public ones come first, or scope the private repo to internal artifacts only.

**Out-of-memory during Maven build**
Increase the Maven heap:

```bash
export MAVEN_OPTS="-Xmx4g"
```
 
