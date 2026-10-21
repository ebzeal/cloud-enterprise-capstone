# TechPathway — Enterprise Capstone Project

A three-service e-commerce application for practicing containerization and Kubernetes deployment. The **Techpathway BothCamp** storefront supports product browsing and checkout, an administration dashboard manages shop data, and separate services handle order notifications and warehouse fulfillment.

## Architecture

```text
Customer / administrator
          |
          v
  weekly-call (:5111)
  Storefront + admin
          |
          +-- HTTP --> action-messages (:5001)
          |            Confirmation emails + tracking numbers
          |
          +-- HTTP --> techpathway-warehouse (:5002)
                       Inventory + fulfillment board
```

Orders are confirmed automatically when placed through checkout or the admin order form. The main app calls both supporting services; each maintains its own data. Warehouse orders progress through `received → picking → packed → shipped`.

| Service | Directory | Local URL | Data storage |
| --- | --- | --- | --- |
| Storefront and admin | `weekly-call/` | [Storefront](http://localhost:5111/store), [admin](http://localhost:5111/) | SQLite by default; optional MySQL/RDS |
| Notifications | `action-messages/` | [Message dashboard](http://localhost:5001/) | SQLite audit log |
| Warehouse | `techpathway-warehouse/` | [Fulfillment board](http://localhost:5002/) | SQLite inventory and orders |

All services use Python and Flask, include a Dockerfile with Gunicorn, and expose `GET /health`. The main app also supports optional S3 image storage. See [ARCHITECTURE.md](ARCHITECTURE.md) for the service contracts and AWS background; some setup and repository-layout details in that document predate this checkout.

## Repository layout

```text
.
├── weekly-call/             # Storefront, admin, static assets, and EC2 helpers
├── action-messages/         # Order notification API and dashboard
├── techpathway-warehouse/   # Inventory API and fulfillment dashboard
├── kubernetes/              # Namespace, configuration, deployments, services, HPA
├── tp008-backup/             # Backup deployment manifest
└── ARCHITECTURE.md          # Detailed architecture notes
```

## Run locally

Use Python 3.11 (the version used by the container images), pip, and three terminal windows. Run each block from the repository root in its own terminal. No AWS account or SMTP server is needed for the default demo.

**1. Start notifications**

```bash
cd action-messages
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python app.py
```

**2. Start the warehouse**

```bash
cd techpathway-warehouse
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python app.py
```

**3. Start the storefront and admin**

```bash
cd weekly-call
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
export SECRET_KEY="$(python -c 'import secrets; print(secrets.token_hex(32))')"
python app.py
```

The apps initialize their SQLite databases on startup; the storefront and warehouse seed eight demo products. With `SMTP_HOST` unset, notifications are logged in demo mode instead of sending email. These commands start Flask development servers for local use.

This checkout does not include a Compose file, `run-local.sh`, `push-to-ecr.sh`, or `.env.example`. Use the commands here rather than the older helper references in the service documentation. Keep local virtual environments and generated database files out of commits.

### Verify the order flow

1. Check the services:

   ```bash
   curl -fsS http://localhost:5111/health
   curl -fsS http://localhost:5001/health
   curl -fsS http://localhost:5002/health
   ```

2. Open the storefront, add a product to the cart, and complete checkout with demo customer details.
3. Confirm the order appears in the admin dashboard and warehouse board.
4. Open the message dashboard to view its confirmation and tracking number.

If an order succeeds but a supporting dashboard has no entry, check that service's terminal and the main app logs. Service calls are synchronous with a short timeout and failures are logged without rejecting the order; there is no durable queue or automatic delivery retry.

## Configuration

The applications load environment variables, including a service-local `.env` when present. `.env` is ignored by Git. Restart a service after changing its configuration.

| Service | Variables | Purpose |
| --- | --- | --- |
| Main app | `SECRET_KEY` | Set a private session-signing key outside local demos. |
| Main app | `NOTIFICATION_SERVICE_URL`, `WAREHOUSE_SERVICE_URL` | Backend URLs; default to `http://localhost:5001` and `http://localhost:5002`. In containers, use reachable service DNS names. |
| Main app | `NOTIFICATION_PUBLIC_URL`, `WAREHOUSE_PUBLIC_URL` | Browser-facing dashboard links; default to the corresponding backend URLs. |
| Main app | `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_DB` | Configure MySQL/RDS instead of local SQLite. Provision and initialize the database before use. |
| Main app | `S3_BUCKET`, `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Optional S3 integration; the current implementation checks the supplied access-key format before enabling S3. |
| Notifications | `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD` | Enable SMTP delivery; port defaults to `587`. An unset host selects demo mode. |
| Notifications | `FROM_EMAIL`, `FROM_NAME`, `ALWAYS_CC_EMAIL` | Sender and confirmation CC settings. Review the built-in CC recipient before enabling real email. |
| Main app and notifications | `SECRETS_MANAGER_SECRET_NAME`, `AWS_REGION` | Optional AWS Secrets Manager JSON configuration; fetched values override local environment values. |

## Build container images

Run from the repository root with Docker installed:

```bash
docker build -t techpathway/weekly-call:local ./weekly-call
docker build -t techpathway/action-messages:local ./action-messages
docker build -t techpathway/warehouse:local ./techpathway-warehouse
```

To deploy to a cluster, tag and push each image to your registry, then update the deployment image references. The checked-in manifests currently reference a specific Amazon ECR account in `us-east-1`; replace those references with images your cluster can pull.

## Deploy to Kubernetes

Prerequisites: a configured cluster, `kubectl` pointing to the intended context, and access to the three container images. Review image references and [the ConfigMap](kubernetes/configmap.yaml) first. Create the namespace and required Secret using your own values:

```bash
kubectl apply -f kubernetes/namespace.yaml

kubectl -n techpathway create secret generic techpathway-secrets \
  --from-literal=SECRET_KEY="$(python3 -c 'import secrets; print(secrets.token_hex(32))')" \
  --from-literal=FROM_EMAIL='orders@example.com' \
  --from-literal=FROM_NAME='TechPathway Team' \
  --from-literal=ALWAYS_CC_EMAIL='demo@example.com'

kubectl apply -f kubernetes/configmap.yaml
kubectl apply -f kubernetes/action-messages.yaml
kubectl apply -f kubernetes/warehouse.yaml
kubectl apply -f kubernetes/weekly-call.yaml

kubectl -n techpathway get pods,services
kubectl -n techpathway rollout status deployment/weekly-call
```

The Secret command is for initial setup and fails if the Secret already exists. `kubernetes/secret.yaml` is ignored by Git; keep credentials out of version control. The supplied notification deployment does not configure SMTP, so it runs in demo mode.

Services are `ClusterIP`. For a local demonstration, run these in separate terminals and use the local URLs above:

```bash
kubectl -n techpathway port-forward service/weekly-call 5111:5111
kubectl -n techpathway port-forward service/action-messages 5001:5001
kubectl -n techpathway port-forward service/techpathway-warehouse 5002:5002
```

Set browser-facing dashboard URLs in the main deployment if needed; its current manifest only imports the two backend URLs from the ConfigMap. `kubernetes/ingress.yaml` is empty, so external routing requires additional configuration.

### Persistence and scaling

The supplied deployments have one replica each and no persistent volume mounts. SQLite files inside containers can be lost when pods are replaced. Add persistent storage before relying on stored orders or inventory.

The optional [HPA manifest](kubernetes/hpa.yaml) scales the main app from 2 to 10 replicas at a 60% CPU target and requires the cluster's resource metrics API. Configure a shared MySQL/RDS database before applying it: the current main deployment does not inject MySQL settings, and separate SQLite replicas would hold different data. Keep notification and warehouse deployments at one replica until their storage supports the intended scaling model.

The dashboards and service APIs have no shared authentication. Add access controls before exposing them publicly.

## Project references

- [Enterprise Capstone Project on Notion](https://app.notion.com/p/Enterprise-Capstone-Project-3a9387dba77280f38691e96a69f4368e)
- [Architecture and service contracts](ARCHITECTURE.md)
- [Main application notes](weekly-call/README.md)

The external references could not be retrieved when this README was prepared. The implementation and checked-in manifests are the basis for the instructions above.
