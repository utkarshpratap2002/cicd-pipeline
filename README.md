# Implementing CI/CD Pipeline using Kubernetes

### Sample application is deployed using the Kubernetes. 

```
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

### Automated deployment using `helm`.

```
helm install mywebsite ./mywebsite-chart
```

### GitHub Setup

```
# initialise
git init

# add to staging area
git add .

git commit -m "version:v1"

```

### Adding GitHub action

Create .github/workflows and write the yaml file that will define the actions for every commit that you'll make.

```
name: CI
permissions:
  contents: write

on:
  push:
    branches:
      - main
    paths-ignore:
      - 'helm/**'
      - README.md

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      - name: Setup docker build
        uses: docker/setup-buildx-action@v1

      - name: Login do DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/mywebsite:${{ github.run_id }}
          platforms: linux/amd64,linux/arm64

  update-helm-chart:
    runs-on: ubuntu-latest
    needs: push
    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      - name: Update tag
        run: |
          sed -i 's/tag: .*/tag: "${{github.run_id}}"/' helm/mywebsite-chart/values.yaml

      - name: Commit and push changes
        run: |
          git config --global user.email "itsutkarshpratapon@gmail.com"
          git config --global user.name "Utkarsh Pratap Singh"
          git add helm/mywebsite-chart/values.yaml
          git commit -m "Update tag in Helm chart"
          git push
```

### Cut down the Human Intervention

Create ArgoCD namespace using kubectl to make your application available outside the cluster.  

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all the pods are running, check whether the pods are running, it might take a minute. 

```
kubectl get pods -n argocd
```

Once all are running, we need to change argocd-server type from *ClusterIP* to *NodePort*. Check argocd-server from the services and edit the *TYPE*.

```
kubectl get service -n argocd
kubectl edit service argocd-server -n argocd
```

Change the *type* to NodePort and check for the change using `kubectl get service -n argocd`. Now the argocd-server is running, use `minikube service argocd-server -n argocd` to see the running service. 

You would require a username *admin* and password will be available inside argocd secrets. 

```
kubectl get secret -n argocd
kubectl edit secret argocd-initial-admin-secret -n argocd
```

Copy from the *password* and decode it in Base64. 

```
echo <base64-password> | base64 --decode
```

Now log into the argocd user interface with username *admin* and decoded password. Create a new application, enter *application name*, copy the *github* link and it will automatically detect the 

