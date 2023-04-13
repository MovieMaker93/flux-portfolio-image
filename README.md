# Flux-portfolio-image

Flux is an open-source project that provides GitOps for both apps and infrastructure.  
The image update feature, provided by Flux CD, automatically updates and reconciles the new version of the app (more info [here](https://fluxcd.io/docs/guides/image-update/)).  
This custom implementation provides a straightforward way to simplify a classic CD process when the container registry receives the latest tag image (CI process [here](https://github.com/MovieMaker93/hugo-arm-site)).

## Prerequisites:
1) Kubernetes cluster version 1.16 or newer
2) Kubectl version 1.18 or above
3) FluxCd installed with image-reflector-controller,image-automation-controller

## Implementation

This custom workflow automatically updates the image tag in the deploy manifest ( simple Hugo app ).
The core concepts about the flux image update are:
1) Define an image repository
2) Define an image policy

The **image repository** tells Flux which container registry to scan and the time interval:  
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
The **image policy** tells Flux which policy to use when filtering tags  
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
The above policy checks the timestamp associated with the tag image version; if the timestamp version in the Dockerhub repository is greater than the image tag used in the deploy manifest, Flux will push the new tag image to the deploy manifest.

The next step is to instruct Flux which image to update and where, so in the deploy manifest,I've defined this label "{"$imagepolicy": "flux-system:hugoapp"}":  
deploy.yaml
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
  ```
In the end, you have to create an **ImageUpdateAutomation** to tell Flux which Git repository to write image updates to:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: furiafc93@gmail.com
        name: moviemaker
      messageTemplate: '{{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: main
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  update:
    path: ./hugo-app-manifests
    strategy: Setters
 ```  
 
## Final Consideration

The above example is one of the multitudes of use cases you can think about this feature.  
For example, you can combine this feature with a CD process with Helm Chart or Kustomitazion.  
So, the image update lets me easily update my blog content with a straightforward process that gives you all the benefits of GitOps in minutes. 
