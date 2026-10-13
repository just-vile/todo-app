# Todo-list — Setup & Deployment Guide

A three-tier Todo application:

| Tier | Service | Image | Port | Notes |
|------|---------|-------|------|-------|
| Frontend | `frontend` | React (Vite) built, served by **nginx** | `3000` | nginx reverse-proxies `/api/` → `backend` |
| Backend | `backend` | **Express** (Node 20) | `5000` | REST API at `/api/todos` |
| Database | `db` | **MongoDB** | `27017` | data persisted in a named volume |

Request flow: **browser → frontend (nginx :3000) → `/api/` proxied to backend :5000 → MongoDB :27017**.
The React app calls the relative path `/api` ([frontend/src/App.jsx](frontend/src/App.jsx)); nginx does the proxying ([frontend/nginx/nginx.conf](frontend/nginx/nginx.conf)); the backend reads `MONGO_URI` from [backend/.env](backend/.env).

---

## 1. Prerequisites

- **Docker Engine 20.10+** on each host.
- For the Swarm deployment: **three Linux VMs** (1 manager + 2 workers) that can reach each other on:
  - `2377/tcp` — cluster management
  - `7946/tcp` + `7946/udp` — node-to-node communication
  - `4789/udp` — overlay network (VXLAN) traffic
- Node.js is **not** required on the host — everything builds inside Docker.

---

## 2. Local development (single host, Docker Compose)

Compose builds the images from source, so this is the fastest way to run everything on one machine.

```bash
docker compose up --build
```

Then open **http://localhost:3000**. Tear down with `docker compose down` (add `-v` to also drop the Mongo data).

> Compose is for local dev only. Swarm (`docker stack deploy`) ignores `build:`, so production deployment uses pre-built images — see below.

---

## 3. Production deployment (Docker Swarm, multi-node)

Deploys with [docker-stack.yml](docker-stack.yml). Because Swarm schedules tasks on any node, the images must come from a **registry every node can reach**, and that registry is served over plain HTTP, so every node must be told to trust it.

Run all commands on the **manager** unless noted.

### 3.1 Initialize the swarm

```bash
docker swarm init --advertise-addr <MANAGER_IP>
# copy the "docker swarm join ..." command it prints and run it on each worker
docker node ls        # all 3 nodes Ready; manager shows "Leader"
```

Find the manager IP (used everywhere below as `MANAGER_IP`):

```bash
hostname -I
```

### 3.2 Start a local registry

```bash
docker service create --name registry --publish published=5000,target=5000 registry:2
```

### 3.3 Trust the insecure registry on **every** node (manager + both workers)

Edit `/etc/docker/daemon.json` on each node, adding `insecure-registries` alongside any existing keys:

```json
{
  "insecure-registries": ["MANAGER_IP:5000"]
}
```

Use the manager's IP (not `localhost`). Then on each node:

```bash
python3 -m json.tool /etc/docker/daemon.json   # validate JSON
sudo systemctl restart docker
docker info | grep -A2 "Insecure Registries"   # confirm it took
```

Skipping this causes `http: server gave HTTP response to HTTPS client` on push/pull.

### 3.4 Build, tag, and push the images

```bash
export REGISTRY=MANAGER_IP:5000
docker build -t $REGISTRY/todo-frontend:latest ./frontend
docker build -t $REGISTRY/todo-backend:latest  ./backend
docker push $REGISTRY/todo-frontend:latest
docker push $REGISTRY/todo-backend:latest
```

### 3.5 Deploy the stack

```bash
docker stack deploy -c docker-stack.yml todo
```

`${REGISTRY:-localhost:5000}` in the stack file picks up your exported `REGISTRY`, so workers pull `MANAGER_IP:5000/todo-...`.

### 3.6 Verify

```bash
docker stack services todo        # expect 2/2, 2/2, 1/1
docker service ps todo_frontend   # shows which node each replica landed on
docker service logs todo_backend  # look for "✅ MongoDB connected"
```

Open **http://<any-node-IP>:3000** — the swarm **ingress** routing mesh forwards to a frontend replica wherever it runs.

---

## 4. Network segmentation

The stack uses three overlay networks to isolate the tiers. Isolation is enforced by *which services share a network* — a service can only resolve/reach another if they share one.

| Network | Attached services | `internal`? | Purpose |
|---------|-------------------|-------------|---------|
| `frontend-net` | frontend | no | Edge tier; the only net with a gateway to the outside |
| `backend-net`  | frontend, backend | **yes** | frontend ⇄ backend |
| `db-net`       | backend, db | **yes** | backend ⇄ db |

Consequences:

- **frontend and db never share a network** → the frontend cannot reach the database at all.
- **backend** is the single chokepoint straddling `backend-net` and `db-net`.
- `backend-net` and `db-net` are `internal: true`, so the entire private tier (backend + db) has **no egress to the internet**. Only `frontend-net` has a gateway, and the database has neither inbound (no published port) nor outbound route.

> If the backend ever needs to call an external service at runtime, remove `internal: true` from `backend-net` (or attach the backend to a non-internal net) — otherwise it fails with "no route to host."

Verify the isolation:

```bash
FID=$(docker ps -qf name=todo_frontend | head -1)
docker exec $FID getent hosts backend   # returns an IP  -> allowed
docker exec $FID getent hosts db          # returns nothing -> blocked

BID=$(docker ps -qf name=todo_backend | head -1)
docker exec $BID getent hosts db           # returns an IP  -> allowed
docker exec $BID sh -c 'wget -qT3 -O- http://example.com'  # fails -> no egress
```

---

## 5. Updating after a code change

Rebuild and push the changed image, then roll the service:

```bash
docker build -t $REGISTRY/todo-backend:latest ./backend
docker push $REGISTRY/todo-backend:latest
docker service update --image $REGISTRY/todo-backend:latest todo_backend
```

The stack's `update_config` rolls one replica at a time (`order: start-first`) for zero-downtime updates.

---

## 6. Teardown

```bash
docker stack rm todo            # remove the app (named volume & its data remain)
docker service rm registry      # remove the registry
docker swarm leave --force      # on each node, to dismantle the swarm
```

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `http: server gave HTTP response to HTTPS client` | node doesn't trust the registry | Add it to `insecure-registries` and restart Docker (step 3.3) |
| `No such image` / task stuck in `Preparing` | worker can't pull from registry | Check the image tag uses `MANAGER_IP:5000`, not `localhost`; confirm the registry service is running |
| Backend logs `MongoDB connection error` | db not reachable | Confirm backend and db share `db-net`; check `docker service ps todo_db` |
| Frontend loads but todos fail | nginx can't reach backend | Confirm frontend and backend share `backend-net`; service name must stay `backend` |
| Replicas stay at `0/2` | image pull or constraint failure | `docker service ps --no-trunc todo_frontend` to see the error |
