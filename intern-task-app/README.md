## Implementation

### Docker

The application is containerized using `python:3.13-slim` to provide a lightweight Python runtime without additional application dependencies.

The Dockerfile creates a non root user (`appuser`) and runs the application with that user to reduce unnecessary privileges.

The application listens on port `5000` by default, configurable through the `PORT` environment variable.

### Kubernetes

The Kubernetes manifests include:

- A Deployment named `intern-task-app` with two replicas.
- Pod labels and Service selectors using `app: intern-task-app`.
- A ClusterIP Service exposing port `80` and forwarding traffic to container port `5000`.
- Readiness and liveness probes using the `/healthz` endpoint.
- CPU and memory requests and limits for predictable resource allocation.
- `imagePullPolicy: Never` because the image is built locally and is not published to a container registry.

The readiness probe controls whether a Pod receives Service traffic. The liveness probe restarts a container after repeated failed health checks. Both check HTTP availability, but the simple `/healthz` endpoint does not validate external dependencies.

### Local Verification - Docker Desktop Kubernetes

The application was tested using Docker Desktop's built in Kubernetes cluster.

From the `intern-task-app` directory:

```powershell
docker build -t intern-task-app:local .
kubectl apply -f k8s/
kubectl rollout status deployment/intern-task-app --timeout=60s
kubectl get pods -l app=intern-task-app
```

Forward the Service port:

```powershell
kubectl port-forward svc/intern-task-app 8080:80
```

In another terminal, verify the endpoints:

```powershell
curl.exe -f http://localhost:8080/healthz
curl.exe -f http://localhost:8080/
```

Expected responses:

- `/healthz` - `ok`
- `/` - `hello from intern-task`

### Troubleshooting and Improvements

During initial testing, one Pod restarted following repeated liveness probe timeouts.

The Python application originally used `HTTPServer`, which processes requests sequentially. It was changed to `ThreadingHTTPServer` to allow concurrent HTTP requests and reduce the possibility of health checks waiting behind other connections.

After rebuilding the Docker image and restarting the Deployment, both Pods remained ready with zero restarts during a five minute observation period. Both HTTP endpoints were also verified successfully through the Kubernetes Service.