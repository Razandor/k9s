# k9s

# Kubernetes Deployment YAML for a web application

# Kubernetes Deployment YAML for a web application
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 4  # Start with 4 replicas to handle peak load
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app-container
        image: your-docker-image:latest # Replace with your image
        resources:
          requests:
            cpu: "0.5"  # Reserve enough CPU for initialization phase
            memory: "128Mi"  # Memory is stable around 128Mi
          limits:
            cpu: "1"  # Limit burst usage to avoid over-allocating
            memory: "128Mi"
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10  # Wait for initialization before probing
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30  # Allow time for app stabilization
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  labels:
    app: web-app
spec:
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP  # Internal service
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2  # Reduce to 2 pods during off-peak hours
  maxReplicas: 6  # Allow up to 6 pods during high load
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # Target 50% CPU utilization
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 3  # Ensure at least 3 pods are available
  selector:
    matchLabels:
      app: web-app
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: your-app.example.com # Replace with your domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
