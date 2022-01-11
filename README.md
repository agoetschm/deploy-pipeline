# Dockerized CI/CD

The idea here is to set up in docker the necessary services to automate the deployment of Helm charts to Kubernetes.

The current possible flow is:
- create some git repositories with their helm chart
- run a Jenkins job which publishes the charts on a Nexus Helm repository
- deploy these charts to Kubernetes with FluxCD

## Setup

Warning: the following steps aim to be reproducible. Nevertheless they depend on the current version of the tools being used, thus they should serve more as inspiration than as an exact recipe.

Other warning: since this is a local setup for experimentation, security is a secondary concern and has been put aside. Do not try to use as it is for any actual project!

Required:
- docker and docker-compose
- gsed (GNU sed)
- kind
- fluxcd

### Gitea

```
docker-compose up gitea
docker-compose exec gitea sh -c 'gitea admin user create --username developer --password developer --email dev@example.com --admin --must-change-password=false'
docker-compose exec gitea sh -c 'gitea admin user create --username jenkins --password jenkins --email jenkins@example.com --must-change-password=false'
```

- `http://localhost:3000`
- add your ssh key to user `developer` (`cat ~/.ssh/id_ed25519.pub`)
- create org `lt`
- create repos `ingest`, `converter`, `forwarder` and `pipeline` with owner `lt`
- create team `ci` in `lt` with admin access
- add user `jenkins` to team `ci`
- add all repos to team `ci`

```
ssh-keyscan localhost >> ~/.ssh/known_hosts
mkdir -p git
REPOS="ingest converter forwarder pipeline"
for REPO in $(echo "$REPOS")
do
  cp -r example/odc/$REPO git
  cp example/odc/Jenkinsfile.release git/$REPO
  cd git/$REPO
  gsed -i "s/REPOSITORY_PLACEHOLDER/$REPO/g" Jenkinsfile.release
  git init
  git remote add origin ssh://git@localhost:22/lt/$REPO.git
  git add .
  git commit -m "initial commit"
  git push -uf origin master
  cd ...
done
```

The `ingest` repo, for example, now contains a `Jenkinsfile.release` describing a job to push its Helm chart to Nexus.

In case you want to delete a user:
`docker-compose exec gitea sh -c 'gitea admin user delete --username developer'`


### Nexus

- `docker-compose up nexus`
- go to `http://localhost:8081/` and sign in as `admin`
  - `docker-compose exec nexus cat /nexus-data/admin.password`
  - set `admin` as password
  - allow anonymous access, for fluxcd later
- admin, repositories, create repo, helm hosted
  - name: `helm-hosted`

### Jenkins

- `docker-compose up jenkins`
- build an image with the necessary tools to run the jobs
  - `docker build --tag jenkinsagent jenkinsagent`
- `http://localhost:8080`, sign in with `admin:admin` and skip plugin installation
- manage jenkins, configure system, add gitea server
  - `http://gitea:3000`
  - manage hooks
    - scope: system
    - `jenkins:jenkins`
- new item, organization folder
  - name: `release`
  - repository source: gitea
  - owner `lt`
  - add credentials again
  - pipeline jenkinsfile: `Jenkinsfile.release`
- check that the gitea org scan sees the repos

Triggering the `ingest` release should publish its chart to Nexus. Same for `converter` and `forwarder`.

### kind (Kubernetes in Docker)

- https://kind.sigs.k8s.io/docs/user/quick-start/
- `kind create cluster`
- `watch kubectl get pods` to monitor the progress

### FluxCD

- https://fluxcd.io/docs/installation/
- https://fluxcd.io/docs/get-started/
- create `flux` repo on gitea with owner `lt`
```
cp -r example/flux git
cd git/flux
git init
git remote add origin ssh://git@localhost:22/lt/flux.git
git add . ; git commit -m "initial commit"; git push -uf origin master
cd ...
```
- create ssh key for flux
  - `mkdir ssh; cd ssh; ssh-keygen -t ed25519 -f key`
  - add `cat key.pub` to gitea `developer` user
- bootstrap flux
  - check /etc/hosts for kubernetes.docker.internal
  - `ssh-keyscan kubernetes.docker.internal >> ~/.ssh/known_hosts`
  - `flux bootstrap git --url=ssh://git@kubernetes.docker.internal:22/lt/flux --private-key-file=ssh/key --branch=master`
  - `flux get kustomizations --watch` to monitor progress

You should now see the Nexus `HelmRepository` as `READY` when checking `kubectl get HelmRepository -n flux-system`.

### Deploy pipeline on cluster

For this step the `pipeline` chart needs to be released.

- add another organization folder `subcharts update` (similar to `release`) targetting pipeline files `Jenkinsfile.depsupdate`
- the Jenkins job should run after the scan and create a pull request 
- merge the pull request in Gitea
- run the release job for the `pipeline`
- the `pipeline` `HelmRelease` in the `flux` repo deploy the latest version of the chart to Kubernetes (`kubectl get HelmRelease`)

You should now see pods running when checking `kubectl get pods`.
