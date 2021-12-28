# Local CI/CD setup

### gitea

```
docker-compose up git
docker exec $(docker ps -qf "name=gitea") sh -c 'gitea admin user create --username developer --password password --email dev@example.com --admin --must-change-password=false'
docker exec $(docker ps -qf "name=gitea") sh -c 'gitea admin user create --username jenkins --password password --email jenkins@example.com --access-token' # keep this access token somewhere
```

- `localhost:3000`
- create org `lt`
- create repo `ingest` with owner `lt`
- create team `ci` in `lt`
- add user `jenkins` to team `ci`
- add repo `ingest` to team `ci`

```
cd git; mkdir ingest; cd ingest; cp ../../Jenkinsfile .
git init
git add .
git commit -m "first commit"
git remote add origin http://localhost:3000/lt/ingest.git
git push -u origin master
```

`docker exec $(docker ps -qf "name=gitea") sh -c 'gitea admin user delete --username developer'`

### jenkins

`https://plugins.jenkins.io/gitea/`

```
docker-compose up jenkins # wait until it's ready
docker compose exec jenkins jenkins-plugin-cli --plugins gitea branch-api workflow-multibranch workflow-aggregator docker-plugin docker-workflow
# restart jenkins
```
- `http://localhost:8080`
- login with `docker-compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`, skip plugin installation
- manage jenkins, configure jenkins, gitea server
  - `http://git:3000`
  - manage hooks
    - gitea personal access token
    - scope: system
    - paste `jenkins` gitea user access token
- new item, organization folder
  - repository source: gitea
  - owner `lt`
  - add token again
- check that the gitea org scan sees the ingest repo
- half soved issue: to run the jenkins job in a docker container, docker need to be available in the jenkins image
  - https://stackoverflow.com/questions/47854463/docker-got-permission-denied-while-trying-to-connect-to-the-docker-daemon-socke
  - https://forums.docker.com/t/docker-client-installed-in-a-docker-container-cant-access-docker-host/22679


### nexus

- `http://localhost:8081/`
- `docker-compose exec nexus cat /nexus-data/admin.password`
- set `password` as password
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
