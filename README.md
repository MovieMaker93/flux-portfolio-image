# Flux-portfolio-image

Flux is a opensource project that implements GitOps behaviour.  
My flux customization is based on the official image update guide that you can find on [flux](https://fluxcd.io/docs/guides/image-update/).  
My scope is to create a simple CI/CD flow for my blog and portfolio site ([portfolio](https://github.com/MovieMaker93/hugo-arm-site)).  
Basically I had the necessity to update my docker image in the deploy manifest everytime tag image version was updated,
so I created this image update scanning with flux.  
This is my deploy manifest used for my blog/portfolio site:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hugo-app
  name: hugo-app
  namespace: hugosite
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hugo-app
  template:
    metadata:
      labels:
        app: hugo-app
    spec:
      containers:
        - name: hugo-app
          image: alfonsofortunato/hugo-app:master-45ff4e9f-1625931299 # {"$imagepolicy": "flux-system:hugoapp"}
          ports:
            - containerPort: 80
              name: http
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hugo-app
  name: hugo-app
  namespace: hugosite
spec:
  ports:
    - name: hugo-app
      port: 80
      targetPort: http
  selector:
    app: hugo-app
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/ingress.class: "traefik"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.tls: "true"
    ingress.kubernetes.io/redirect-regex: "^https://(www.blogfolio.org|(blog.)?blogfolio.org)/?(.*)"
    ingress.kubernetes.io/redirect-replacement: "https://blogfolio.org/$3"
  name: hugo-app
  namespace: hugosite
spec:
  rules:
    - host: blogfolio.org
      http:
        paths:
          - backend:
              serviceName: hugo-app
              servicePort: 80
            path: /
    - host: www.blogfolio.org
      http:
        paths:
          - backend:
              serviceName: hugo-app
              servicePort: 80
    - host: blog.blogfolio.org
      http:
        paths:
          - backend:
              serviceName: hugo-app
              servicePort: 80
  tls:
    - hosts:
        - blogfolio.org
        - www.blogfolio.org
        - blog.blogfolio.org
      secretName: blogfolio-tls
  ```
I have defined an `ImageRepository` to tell Flux which dockerhub repository to use and the scanning interval:
```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: hugoapp
  namespace: flux-system
spec:
  image: alfonsofortunato/hugo-app
  interval: 5m0s
```
I have also created an `ImagePolicy` to tell Flux which policy to use when filtering tags:
```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: hugoapp
  namespace: flux-system
spec:
  filterTags:
    pattern: '^master-[a-fA-F0-9]+-(?P<ts>.*)'
    extract: '$ts'  
  imageRepositoryRef:
    name: hugoapp
  policy:
    numerical:
      order: asc
```
The above policy check the timestap associated with the tag image version, if the timestap version in the dockerhub repository is greater then the image tag used in the deploy manifest, flux will push the new tag image to deploy manifest.


