


# This is an example of a traefik auth middleware to protect a service
# As Traefik do not allow Cross Reference between middlewares and services, we need to create a new middleware for each service
# By following below example, we can create a new middleware for each service and protect it with the forward auth service 

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: github-auth-middleware
  namespace: forward-auth
spec:
  forwardAuth:
    address: http://github-forward-auth-service.forward-auth.svc.cluster.local:4181/verify # This is the address of the forward auth service
    trustForwardHeader: true
    authResponseHeaders:
      - X-Forwarded-User
      - X-Forwarded-Email

```

# This is an example of ingress route with the middleware

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: phpmyadmin
  namespace: mysql
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`phpmyadmin.web.himelrana.eu.org`)
      kind: Rule
      services:
        - name: phpmyadmin
          port: 80
      middlewares:
        - name: github-forward-auth-middleware # This is the name of the middleware
          namespace: mysql
  tls:
    secretName: phpmyadmin-tls # This is the name of the certificate

  ```