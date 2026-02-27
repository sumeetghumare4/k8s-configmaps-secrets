# k8s-configs

A minimal Express‑based backend with accompanying Kubernetes manifests.
The emphasis is on learning how ConfigMaps, Secrets and Deployments
work; the application itself simply echoes environment variable values.

---

## 📁 Repository layout

```
.
├── Dockerfile
├── index.ts           # simple Express server reading env vars
├── package.json
├── tsconfig.json
├── backend/           # (empty placeholder)
└── ops/
    ├── config.yml     # ConfigMap for non-sensitive settings
    ├── secrets.yml    # Secret with encoded values
    └── deployment.yml # Deployment manifest consuming config/secret
```

## 🧩 Backend overview

The Node/Bun backend returns the values of `PORT` and
`DATABASE_URL` environment variables at the root path:

```ts
app.get("/", (req, res) => {
  res.json({
    db: process.env.DATABASE_URL,
    port: process.env.PORT
  });
});

app.listen(process.env.PORT);
```

It is built from the `Dockerfile` and published as
`sumeetghumare/todo-app-test:1` in the manifests.

## 🚀 Kubernetes concepts

### ConfigMap

A `ConfigMap` holds non‑sensitive configuration data as key/value pairs.
`ops/config.yml` defines one:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  port: "3000"
```

The Deployment references it with `valueFrom.configMapKeyRef` to inject
`PORT` into the container. ConfigMaps let you change configuration
without rebuilding images.

### Secret

`ops/secrets.yml` stores sensitive strings base64‑encoded:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
data:
  database_url: "cG9zdGdyZXM6Ly8xMjMK"  # base64 of postgres://... 
```

Kubernetes keeps the raw value out of plain manifests and etcd. Use
`echo -n 'value' | base64` when creating secrets, and avoid committing
secrets to git.

### Deployment

`ops/deployment.yml` declares the pod template, selector, and env vars:

```yaml
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: ecom-backend
    spec:
      containers:
      - name: todo-backend
        image: sumeetghumare/todo-app-test:1
        ports:
        - containerPort: 3000
        env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: port
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: database_url
```

Labels drive service discovery and support rolling upgrades or scaling.

### Applying to a cluster

1. **Build & push the image** (use Docker or Bun):

   ```bash
   docker build -t sumeetghumare/todo-app-test:1 .
   docker push sumeetghumare/todo-app-test:1
   ```

2. **Deploy manifests**:

   ```bash
   kubectl apply -f ops/config.yml
   kubectl apply -f ops/secrets.yml
   kubectl apply -f ops/deployment.yml
   ```

3. **Verify**:

   ```bash
   kubectl get configmap,secret,deploy
   kubectl logs deploy/todo-app-backend
   ```

Updating the ConfigMap or Secret and then running
`kubectl rollout restart deploy/todo-app-backend` lets you change the
app's configuration without rebuilding the container.

> 🔒 Secrets are stored encoded and should never appear in Git history.

### Notes & best practices

- `node_modules/` is ignored; never commit dependencies.
- ConfigMaps are for non‑sensitive data; Secrets for credentials, API
  keys, etc.
- Environment variables are a common way to expose config to containers.

## 🛠 Local development

```bash
bun install
PORT=3000 DATABASE_URL=postgres://... bun run index.ts
```

You can also run the app inside the Docker image for parity with the
cluster.

---

*This repository is intended for learning and demonstration of basic
Kubernetes configuration patterns.*
