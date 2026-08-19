# kotlin-fhirpath-server

A Ktor-based server that evaluates [FHIRPath](https://hl7.org/fhirpath/) expressions against FHIR
resources. It implements the
[FHIRPath Lab Server Engine API](https://github.com/brianpos/fhirpath-lab/blob/master/server-api.md)
specification, supporting FHIR versions **R4**, **R4B**, and **R5**.

## Prerequisites

- **Java 21** (the project uses JVM toolchain 21)
- **Gradle** (wrapper included — no separate installation needed)

## API

The server exposes three FHIRPath evaluation endpoints — one per FHIR version — plus a health check:

|            Endpoint            | Method |                                 Description                                 |
|--------------------------------|--------|-----------------------------------------------------------------------------|
| `/`                            | GET    | API overview, endpoint listing, and server version                          |
| `/health`                      | GET    | Health check with current timestamp                                         |
| `/kotlin-fhirpath-config.json` | GET    | [FHIRPath Lab custom engine configuration][custom-config] for local testing |
| `/fhirpath-r4`                 | POST   | Evaluate a FHIRPath expression against an **R4** resource                   |
| `/fhirpath-r4b`                | POST   | Evaluate a FHIRPath expression against an **R4B** resource                  |
| `/fhirpath-r5`                 | POST   | Evaluate a FHIRPath expression against an **R5** resource                   |

[custom-config]: https://github.com/brianpos/fhirpath-lab/blob/develop/docs/custom-configuration.md

### Request

**Content-Type**: `application/fhir+json` or `application/json`

**Body**: A FHIR `Parameters` resource. See the
[input parameters definition](https://github.com/brianpos/fhirpath-lab/blob/master/server-api.md#input-parameters-resource)
for the full parameter specification. Support status in this implementation:

|      Parameter      | Supported |
|---------------------|:---------:|
| `expression`        |     ✅     |
| `resource`          |     ✅     |
| `context`           |     ✅     |
| `variables`         |     ✅     |
| `terminologyserver` |     ❌     |

**Example request body:**

```json
{
  "resourceType": "Parameters",
  "parameter": [
    {
      "name": "expression",
      "valueString": "name.family"
    },
    {
      "name": "resource",
      "resource": {
        "resourceType": "Patient",
        "name": [{ "family": "Smith", "given": ["John"] }]
      }
    }
  ]
}
```

### Response

A successful evaluation returns HTTP `200` with a FHIR `Parameters` resource containing the results
and debug trace information.

Validation errors return HTTP `400` with an `OperationOutcome`. Unexpected server errors return HTTP
`500` with an `OperationOutcome`.

## Versioning

The build derives the server version from `git describe --tags --always --dirty` (see
`build.gradle.kts`). The `/` endpoint reports this version. Every deployment therefore identifies
the release or commit that produced it.

The build removes the leading `v` from the tag name:

|         Build point          |                   Reported version                    |
|------------------------------|-------------------------------------------------------|
| On the exact tag `v1.2.3`    | `1.2.3`                                               |
| 4 commits after tag `v1.2.3` | `1.2.3-4-gabc1234` (commit count and abbreviated SHA) |
| Before the first tag exists  | The abbreviated commit SHA                            |
| With uncommitted changes     | The same value, plus the suffix `-dirty`              |
| When git is unavailable      | `0.0.0-unknown`                                       |

### Create a release

To release a new version, tag the commit and push the tag:

```bash
git tag v1.2.3
git push origin v1.2.3
```

Every later build reads the new tag, including the Docker build. You do not need to edit a version
number anywhere in the project. You can also turn the tag into a GitHub Release, but the version
does not depend on it.

## Deployment

### Local

Use the Gradle wrapper to build and run the server:

|          Task           |                          Description                          |
|-------------------------|---------------------------------------------------------------|
| `./gradlew run`         | Run the server locally                                        |
| `./gradlew test`        | Run the test suite                                            |
| `./gradlew build`       | Compile and assemble the project                              |
| `./gradlew buildFatJar` | Build a self-contained executable JAR (`fhirpath-server.jar`) |

The server starts on port `8080` by default. Set the `PORT` environment variable to override:

```bash
PORT=9090 ./gradlew run
```

When the server starts successfully you will see:

```
2024-12-04 14:32:45.584 [main] INFO  Application - Application started in 0.303 seconds.
2024-12-04 14:32:45.682 [main] INFO  Application - Responding at http://0.0.0.0:8080
```

#### Running the fat JAR directly

```bash
./gradlew buildFatJar

java -jar build/libs/fhirpath-server.jar
```

### Docker

A [Dockerfile](deploy/Dockerfile) and [docker-compose.yml](deploy/docker-compose.yml) are included
under `deploy/`. Build and run with Docker Compose:

```bash
docker compose -f deploy/docker-compose.yml up --build
```

Or build and run the image directly (the build context is the repo root):

```bash
docker build -f deploy/Dockerfile -t fhirpath-server .
docker run -p 8080:8080 fhirpath-server
```

The Ktor Gradle plugin also provides Docker tasks as an alternative to the Dockerfile:

|                  Task                   |                  Description                   |
|-----------------------------------------|------------------------------------------------|
| `./gradlew buildImage`                  | Build a Docker image from the fat JAR          |
| `./gradlew publishImageToLocalRegistry` | Publish the image to the local Docker registry |
| `./gradlew runDocker`                   | Build the image and run it as a container      |

### Application Server (Production)

The server is packaged as a Docker image and run on any host with Docker installed. Deployment is
currently manual — there is no CI/CD pipeline.

#### What you need before deploying

- Docker installed on the target host
- A [Docker Hub](https://hub.docker.com) account. Images are published to the public repository
  [`nawitech/kotlin-fhirpath`](https://hub.docker.com/r/nawitech/kotlin-fhirpath). Because the
  repository is public, the host pulls without authenticating.

The image name and tag are configurable via the `IMAGE` and `TAG` environment variables (defaults:
`docker.io/nawitech/kotlin-fhirpath` and `latest`). Set `TAG` to publish and deploy a specific
version instead of the moving `latest` tag — for example `export TAG=v1.2.0`. See
[`deploy/.env.example`](deploy/.env.example).

#### Build and push the image

Publishing needs write access to the Docker Hub repository. Authenticate once with an **access
token** (Docker Hub → *Account Settings → Security → New Access Token*, with *Read & Write* scope) —
not your account password:

```bash
docker login -u nawitech        # paste the access token when prompted for a password
```

Then build and push. The first push to a public repository creates it automatically:

```bash
docker build -f deploy/Dockerfile -t nawitech/kotlin-fhirpath:${TAG:-latest} .
docker push nawitech/kotlin-fhirpath:${TAG:-latest}
```

Or use Docker Compose, which reads the image name/tag from `deploy/.env` (or `IMAGE`/`TAG` in the
environment) and tags the build for you:

```bash
docker compose -f deploy/docker-compose.yml build
docker compose -f deploy/docker-compose.yml push
```

#### Deploy to the host

On the target host, pull the image and start the container. The repository is public, so no
`docker login` is needed. Bind the container to **loopback** (`127.0.0.1`) so the only way in is
through nginx, which terminates TLS (set up in the next section):

```bash
docker pull nawitech/kotlin-fhirpath:${TAG:-latest}
docker run -d --restart unless-stopped \
  -p 127.0.0.1:8080:8080 \
  --name fhirpath-server \
  nawitech/kotlin-fhirpath:${TAG:-latest}
```

Open ports **80 and 443** in the host firewall/security group (for nginx).
Port `8080` stays closed to the outside — it's bound to loopback.

#### Exposure: nginx + TLS (HTTPS)

Clients reach the server through nginx, which terminates TLS and reverse-proxies
to the loopback container. The server is exposed **by IP address** with a
CA-issued [Let's Encrypt IP certificate][le-ip]. Moving to a subdomain later is a
small edit (see the last step).

[le-ip]: https://letsencrypt.org/2026/01/15/6day-and-ip-general-availability

The nginx config lives at
[`deploy/nginx/fhirpath-server.conf`](deploy/nginx/fhirpath-server.conf). It sets
`server_name` to the IP, serves ACME HTTP-01 challenges from `/var/www/certbot`
(so certbot can renew without downtime), and points `ssl_certificate` at the
Let's Encrypt cert.

Three things to know about Let's Encrypt IP certificates before you start:

- **They are short-lived — 6 days.** Let's Encrypt requires the `shortlived`
  profile for IP addresses, so **automated renewal is mandatory** (set up below).
- **certbot's nginx plugin does not support IP addresses.** You obtain the cert
  with the `webroot` plugin and wire nginx to it by hand — `certbot --nginx`
  won't work for an IP.
- You need **certbot ≥ 5.4**, a **public, routable IP**, and **port 80 reachable**
  from the internet for HTTP-01 validation on every renewal.

##### Obtain the IP certificate

With the container running on loopback (per the previous section):

```bash
SERVER_IP=203.0.113.7            # <-- your server's public IP

# 1. Bootstrap: a temporary self-signed cert so nginx can start before the
#    Let's Encrypt cert exists (nginx won't load an ssl_certificate that isn't
#    there yet, and webroot renewal needs nginx already serving port 80).
sudo mkdir -p /etc/nginx/ssl /var/www/certbot
sudo openssl req -x509 -nodes -days 825 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/fhirpath-server.key \
  -out    /etc/nginx/ssl/fhirpath-server.crt \
  -subj   "/CN=${SERVER_IP}" \
  -addext "subjectAltName=IP:${SERVER_IP}"

# 2. Install the site config, substituting the IP placeholder.
sudo cp deploy/nginx/fhirpath-server.conf \
  /etc/nginx/sites-available/fhirpath-server.conf
sudo sed -i "s/__SERVER_IP__/${SERVER_IP}/g" \
  /etc/nginx/sites-available/fhirpath-server.conf
sudo ln -sf /etc/nginx/sites-available/fhirpath-server.conf \
  /etc/nginx/sites-enabled/fhirpath-server.conf

# 3. Temporarily point the ssl_certificate lines at the bootstrap cert, then
#    start nginx so it can serve the ACME challenge on port 80.
sudo sed -i \
  -e "s#/etc/letsencrypt/live/${SERVER_IP}/fullchain.pem#/etc/nginx/ssl/fhirpath-server.crt#" \
  -e "s#/etc/letsencrypt/live/${SERVER_IP}/privkey.pem#/etc/nginx/ssl/fhirpath-server.key#" \
  /etc/nginx/sites-available/fhirpath-server.conf
sudo nginx -t && sudo systemctl reload nginx

# 4. Make sure certbot is recent enough (distro packages are often too old).
certbot --version                # need >= 5.4;  else: sudo snap install --classic certbot

# 5. Request the cert via webroot. Try --staging first to avoid rate limits,
#    then re-run without --staging for the real cert.
sudo certbot certonly --staging \
  --preferred-profile shortlived \
  --webroot --webroot-path /var/www/certbot \
  --ip-address ${SERVER_IP}

# 6. Once issued, point nginx back at the Let's Encrypt cert and reload.
sudo sed -i \
  -e "s#/etc/nginx/ssl/fhirpath-server.crt#/etc/letsencrypt/live/${SERVER_IP}/fullchain.pem#" \
  -e "s#/etc/nginx/ssl/fhirpath-server.key#/etc/letsencrypt/live/${SERVER_IP}/privkey.pem#" \
  /etc/nginx/sites-available/fhirpath-server.conf
sudo nginx -t && sudo systemctl reload nginx

# 7. Verify (no -k needed — the cert is trusted).
curl https://${SERVER_IP}/health
```

The self-signed files under `/etc/nginx/ssl` can be deleted once the CA cert is
live.

##### Automate renewal (required)

IP certs expire in 6 days, and certbot can't reload nginx itself for them, so
register a deploy hook that reloads nginx after each renewal:

```bash
sudo certbot reconfigure --deploy-hook "systemctl reload nginx"

# certbot's systemd timer renews twice daily — well inside the 6-day window.
systemctl list-timers | grep certbot
sudo certbot renew --dry-run
```

##### Later: moving to a subdomain

When a subdomain is ready, point its DNS `A` record at the server, then switch
off the IP cert. For a domain, certbot's nginx plugin *does* work, so this is
nearly automatic:

```bash
FQDN=fhirpath.example.com        # <-- your subdomain

# 1. Point server_name at the FQDN (both server blocks).
sudo sed -i "s/server_name ${SERVER_IP};/server_name ${FQDN};/g" \
  /etc/nginx/sites-available/fhirpath-server.conf
sudo nginx -t && sudo systemctl reload nginx

# 2. Obtain a standard CA cert; certbot rewrites the ssl_* lines for you.
sudo certbot --nginx -d ${FQDN}

# 3. Verify.
curl https://${FQDN}/health
```

#### Deploying an update

```bash
# 1. Build and push a new image (from your build machine)
docker build -f deploy/Dockerfile -t nawitech/kotlin-fhirpath:${TAG:-latest} .
docker push nawitech/kotlin-fhirpath:${TAG:-latest}
# or, using Compose (reads IMAGE/TAG from deploy/.env):
#   docker compose -f deploy/docker-compose.yml build
#   docker compose -f deploy/docker-compose.yml push

# 2. Pull and restart on the host (nginx and the cert are untouched)
docker pull nawitech/kotlin-fhirpath:${TAG:-latest}
docker stop fhirpath-server
docker rm fhirpath-server
docker run -d --restart unless-stopped -p 127.0.0.1:8080:8080 --name fhirpath-server nawitech/kotlin-fhirpath:${TAG:-latest}
```

#### Deploying to a Google Cloud Compute Engine VM

Authenticate the Docker CLI against Artifact Registry, then use the commands above with your
registry path as `<REGISTRY>/<IMAGE>:<TAG>`:

```bash
gcloud auth login
gcloud config set project <PROJECT_ID>
gcloud auth configure-docker <REGION>-docker.pkg.dev
# registry path: <REGION>-docker.pkg.dev/<PROJECT_ID>/<REPO>/fhirpath-server:<TAG>
```

To run commands on the VM remotely:

```bash
# open a shell
gcloud compute ssh <INSTANCE_NAME> --zone <ZONE>

# or run a single command
gcloud compute ssh <INSTANCE_NAME> --zone <ZONE> --command "docker pull ..."
```

## Specification

This server implements the
[FHIRPath Lab Server Engine API](https://github.com/brianpos/fhirpath-lab/blob/master/server-api.md).

Once deployed, one would be able to point [FHIRPath Lab](https://fhirpath-lab.com) to this server to use as an
evaluation engine.
