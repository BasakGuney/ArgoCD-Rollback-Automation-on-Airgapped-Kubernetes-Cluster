# ArgoCD Rollback Automation on Airgapped Kubernetes Cluster

## Purpose
>Enabling auto-rollback to the last "Healthy" commit when the app status is "Degraded".

## Path
>A customized alpine image with ArgoCD CLI and git installed in it will be created. Since our cluster doesn't have Internet access, we will make installations with binaries that we will be download using our host machine and send it to cluster. After the customized image has created, we will create an agent pod that is responsible for rollback. In the agent pod, we will check the ArgoCD status continuously if the status is "Healthy" we will tag this commit as "Healthy" using gi commands. If the app status is "Degraded" we will rollback to "Healthy" tagged commit.


## Steps

### 1. Downloading ArgoCD Binary
On your host machine, download ArgoCD binary from https://github.com/argoproj/argo-cd/releases and send it to a cluster node which has docker in it via scp.

### 2. Creating Custom Alpine Image
Create a Dockerfile in the same directory with the ArgoCD binary.

You can use the following Dockerfile:
```bash
FROM <your-local-repo>/alpine/git:latest

# Set environment variables
ENV ARGOCD_CLI_PATH="/usr/local/bin/argocd"

# Copy ArgoCD binary from the host into the container
COPY argocd ${ARGOCD_CLI_PATH}

# Set executable permissions for the ArgoCD binary
RUN chmod +x ${ARGOCD_CLI_PATH}

# Verify installations
RUN git --version && argocd version --client && argocd login <argocd-server-domain> --username admin --password <password>

# Set working directory
WORKDIR /app

# Override the default entry point
ENTRYPOINT ["/bin/sh"]

# Default command
CMD []

```

```bash
docker build -t <your-local-repo>/alpine-git-argocd .
```
Build the image.
<br>
```bash
docker push <your-local-repo>/alpine-git-argocd 
```
Push the image to your repo.
<br>

### 3. Creating Agent Pod

You can use the following YAML:

```bash
apiVersion: v1
kind: Pod
metadata:
  name: rollback-agent
  labels:
    app: rollback-agent
spec:
  containers:
    - name: rollback-agent
      image: <your-local-repo>/alpine-git-argocd
      command: ["/bin/sh", "-c"]
      args:
        - |
          git config --global user.name <username>
          git config --global user.email <email>
          git config --global credential.helper store
          echo "http://<username>:<password>@<node-ip-where-gitea-placed>%3a<port>" > ~/.git-credentials
          chmod 600 ~/.git-credentials
          git clone <your-app-repo>
          cd python-project
          argocd login <argocd-server-domain> --username <username> --password <password> --insecure;
          APP_NAME=<your-app-name>
          sleep 30;
          while true; do
            STATUS=$(argocd app get $APP_NAME  | awk '/Health Status/ {print $3}')
            echo "Application health status: $STATUS"
            if [ "$STATUS" == "Healthy" ]; then
              git tag -d Healthy
              git push origin --delete Healthy
              git tag Healthy 
              git push origin Healthy
              echo "No rollback needed."
            elif [ "$STATUS" != "Healthy" ] && [ "$STATUS" != "Progressing" ] && [ "$STATUS" != "Missing" ]; then
              echo "Performing rollback..."
              COMMIT_HASH=$(git show Healthy --pretty=format:%H | head -n 1 | cut -c1-7)
              argocd app sync <your-app-name> --prune --force --replace --revision $COMMIT_HASH
            # Sleep for 5 seconds to avoid constant looping
            fi
            sleep 5
          done          
```
<br>
<br>

The following lines make credential configuration that helps ta avoid interactive login:

```bash
git config --global user.name <username>
git config --global user.email <email>
git config --global credential.helper store
echo "http://<username>:<password>@<node-ip-where-gitea-placed>%3a<port>" > ~/.git-credentials
chmod 600 ~/.git-credentials
```
<br>

if the current commit is "Healthy" following commands slides the "Healthy" tag to the current commit, by this the we can ensure that we will rollback to the "latest" healthy commit.
 ```bash
 if [ "$STATUS" == "Healthy" ]; then
   git tag -d Healthy
   git push origin --delete Healthy
   git tag Healthy 
   git push origin Healthy
   echo "No rollback needed."
 ```
 <br>

 Following takes the hash of the "Healthy" tagged commit:
 ```bash
 COMMIT_HASH=$(git show Healthy --pretty=format:%H | head -n 1 | cut -c1-7)
 ```
 <br>

 Following command sync the app with the healthy commit and recreates all resources from scratch and prunes the unnecessary ones.
 ```bash
 argocd app sync <your-app-name> --prune --force --replace --revision $COMMIT_HASH
 ```
