# Local CI/CD setup

The idea here is to set up in docker the necessary services to automate the deployment of Helm charts to Kubernetes.

The current possible flow is: 
- create a git repository with a helm chart
- run a Jenkins job which publishes the chart on a Nexus Helm repository
- deploy this chart to Kubernetes with FluxCD

The following steps aim to be reproducible. Nevertheless they depend on the current version of the tools being used, thus they should serve more as inspiration than as an exact recipe.

### gitea

```
docker-compose up git
docker-compose exec git sh -c 'gitea admin user create --username developer --password password --email dev@example.com --admin --must-change-password=false'
docker-compose exec git sh -c 'gitea admin user create --username jenkins --password password --email jenkins@example.com --access-token' # keep this access token somewhere
```

- `http://localhost:3000`
- add your ssh key to user `developer` (`cat ~/.ssh/id_ed25519.pub`)
- create org `lt`
- create repo `ingest` with owner `lt`
- create team `ci` in `lt` with admin access
- add user `jenkins` to team `ci`
- add repo `ingest` to team `ci`

```
mkdir git; cd git; mkdir ingest; cd ingest; cp ../../jenkins/Jenkinsfile .
mkdir charts; cd charts; helm create ingest; cd ..
git init; git add .; git commit -m "first commit"
git remote add origin http://localhost:3000/lt/ingest.git
git push -u origin master
```

The `ingest` repo now contains a `Jenkinsfile` describing a job to push its Helm chart to Nexus.

In case you want to delete a user:
`docker exec $(docker ps -qf "name=gitea") sh -c 'gitea admin user delete --username developer'`


### nexus

- `docker-compose up nexus`
- go to `http://localhost:8081/` and sign in
  - `docker-compose exec nexus cat /nexus-data/admin.password`
  - set `password` as password
  - allow anonymous access, for fluxcd later
- admin, repositories, create repo, helm hosted
  - name: `helm-hosted`



pip3 install nexus3-cli
nexus3 login -U http://localhost:8081 -u admin -p $(cat persistence/nexus/admin.password) --x509_verify
- this cli is not official and doesn't seem to work with the latest version
- needs https://support.sonatype.com/hc/en-us/articles/360045220393-Scripting-Nexus-Repository-Manager-3#how-to-enable
-  but it doesn't support creating helm repos...
- so for it's easier to configure via the ui
  - add helm-hosted repo
  - helm repo add nexus http://localhost:8081/repository/helm-hosted --username admin --password password

https://help.sonatype.com/repomanager3/nexus-repository-administration/formats/helm-repositories


### jenkins

`https://plugins.jenkins.io/gitea/`

- `docker-compose up jenkins`
- `http://localhost:8080`, sign in with `admin:password`
- manage jenkins, configure system, add gitea server
  - `http://git:3000`
  - manage hooks
    - gitea personal access token
    - scope: system
    - paste `jenkins` gitea user access token
- new item, organization folder
  - repository source: gitea
  - owner `lt`
  - use jenkins user in gitea (TODO: user token)
- check that the gitea org scan sees the ingest repo

The `ingest` chart should be published to Nexus by this job.

### kind (Kubernetes in Docker)

- https://kind.sigs.k8s.io/docs/user/quick-start/
- `kind create cluster`
- `watch kubectl get pods` to monitor the progress

### fluxcd

- https://fluxcd.io/docs/installation/
- https://fluxcd.io/docs/get-started/
- create flux repo on gitea with owner `lt` and initial readme (not empty)
- check /etc/hosts for kubernetes.docker.internal
```
cd git; git clone ssh://git@kubernetes.docker.internal:22/lt/flux # also adds to known hosts
cd flux; cp ../../fluxcd/* .
git add .; git commit -m "add nexus and ingest"; git push
```
- create ssh key for flux
  - `mkdir ssh; cd ssh; ssh-keygen -t ed25519 -f key`
  - add `cat key.pub` to gitea `developer` user
- `flux bootstrap git --url=ssh://git@kubernetes.docker.internal:22/lt/flux --private-key-file=ssh/key --branch=master`
  - `flux get kustomizations --watch` to monitor progress
