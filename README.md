# Gitlab CICD

## Run GitLab

```bash
cd ./gitlab
echo "PUBLIC_IP=10.147.17.219" >> .env
docker-compose up -d
docker exec -it gitlab-cicd_gitlab grep 'Password:' /etc/gitlab/initial_root_password && echo 'User: root'
```

## Create Repository

1. Create repository
2. Create Access Token for new user (Settings->Access Token)
3. Create local repository:

```bash
mkdir GitHub/repo; cd GitHub/repo
git init --initial-branch=main
git remote add origin http://<token-name>:<access-token>@<gitlab-address>/root/<repo>.git
# git remote add origin http://domas:TzRy9kMmxpFTxxSAqGkJ@10.147.17.219:8929/root/web.git
git branch --set-upstream-to origin/main
```

## Register CI/CD Runner

1. Get GitLab's `url` and `registration-token` (Settings->CI/CD)
2. Run `docker-compose exec -T runner gitlab-runner register`:

```bash
docker-compose exec -T runner gitlab-runner register -n \
  --url <url> \
  --registration-token <registration-token> \
  --executor docker \
  --docker-image docker:stable \
  --description "my_runner"
```

3. Add insecure registry to docker in `host` (docker is mapped from host to gitlab-runner)

```bash
export DOCKER_REGISTRY=10.147.17.219:5005
sudo rm /etc/docker/daemon.json
sudo tee -a /etc/docker/daemon.json <<EOF
{
  "insecure-registries" : ["$DOCKER_REGISTRY"]
} 
EOF
sudo systemctl restart docker
```

## React Example

Everything below is in `repo` directory

### init

```bash
cd repo
npx create-react-app .
git add .
git commit -m "init"
gi push
```

### tip: add standard version

```bash
yarn add standard-version
```

```json
# ./package.json
{
    ...
    "scripts": {
        ...
        "release": "standard-version",
        "postrelease": "git push -o ci.skip && git push origin v$npm_package_version"
    }
}

```

### build pipeline

```yaml
# ./.gitlab-ci.yml
stages:
  - build

prod_build:
  image: node:14
  stage: build
  script:
    - cd app
    - NODE_ENV=prod
    - yarn install --frozen-lockfile --check-files
    - yarn test --watchAll=false
    - yarn build
  artifacts:
    paths:
      - build/
```

### docker image creation pipeline

```nginx
# ./nginx.conf
server {
  listen 80;
  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html =404;
  }
}
```

```Dockerfile
# ./Dockerfile
FROM nginx:alpine
COPY ./nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

```yaml
# ./.gitlab-ci.yml
stages:
  - build
  - docker
...
docker_build:
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  stage: docker
  only:
    - tags
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY_IMAGE:latest
```

### deploy pipeline

0. connect to deployment machine

1. pull repo

```bash
mkdir GitHub/repo; cd GitHub/repo
git init --initial-branch=main
git remote add origin http://<token-name>:<access-token>@<gitlab-address>/root/<repo>.git
# git remote add origin http://domas:TzRy9kMmxpFTxxSAqGkJ@10.147.17.219:8929/root/web.git
git branch --set-upstream-to origin/main
```

2. setup insecure docker registry

```bash
export DOCKER_REGISTRY=10.147.17.219:5005
sudo rm /etc/docker/daemon.json
sudo tee -a /etc/docker/daemon.json <<EOF
{
  "insecure-registries" : ["$DOCKER_REGISTRY"]
} 
EOF
sudo systemctl restart docker
docker login $DOCKER_REGISTRY -u <token-name> -p <access-token>
```

3. make ssh keys and add public key to deployment machine

```bash
mkdir -p deploy; cd deploy
ssh-keygen -f ./id_rsa -N ""
cat ./deploy/id_rsa.pub >> ~/.ssh/authorized_keys
```

4. add docker-compose.yaml file

```yaml
# ./deploy/docker-compose.yaml
version: "3"
services:
  repo:
    container_name: repo_web
    image: '10.147.17.219:5005/root/web:latest' # change to gitlab ip
    restart: always
    ports:
      - "1234:80"
```

5. add deploy script

```bash
# ./deploy/deploy_prod.sh
export SINGLE_PROD_ADDRESS=<user>@<deploy-machine-ip>
echo $SINGLE_PROD_ADDRESS
ssh -i ./deploy/id_rsa $SINGLE_PROD_ADDRESS << EOF
  cd GitHub/temp/repo/
  git pull
  cd deploy
  docker-compose pull
  docker-compose up --build -d
EOF
```

6. update gitlab config

```yaml
# ./.gitlab-ci
stages:
  - build
  - docker
  - deploy
...
prod_deploy:
  stage: deploy
  script:
    - chmod +x ./deploy/deploy_prod.sh
    - chmod 600 ./deploy/id_rsa
    - mkdir -p ~/.ssh && touch ~/.ssh/known_hosts
    - ssh-keyscan 10.147.17.219 >> ~/.ssh/known_hosts # change to deploy ip
    - sh ./deploy/deploy_prod.sh
  environment:
    name: production
```
