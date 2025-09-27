```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: express-ingress
  namespace: mongodb
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"   # if using cert-manager
spec:
  ingressClassName: traefik
  rules:
  - host: express.web.himelrana.eu.org
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mongo-express
            port:
              number: 80
  tls:
  - hosts:
    - express.web.himelrana.eu.org
    secretName: mongo-express-tls
```