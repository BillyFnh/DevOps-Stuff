# Collection of GitLab Pipelines

## Building Node.js

```yaml
backend-server-image-build:
  image: docker:19.03.1
  stage: build
  variables:
    CI_CONTAINER_IMAGE: <username>/<image-name>:$CI_COMMIT_SHORT_SHA
  before_script:
    - docker login -u <username> -p <password>
  script:
    - docker build -t "$CI_CONTAINER_IMAGE" .
    - docker tag "$CI_CONTAINER_IMAGE" "$CI_CONTAINER_IMAGE"
    - docker tag "$CI_CONTAINER_IMAGE" <username>/<image-name>:latest
    - docker push "$CI_CONTAINER_IMAGE"
  after_script:
    - docker login -u <username> -p <password>
    - docker image rm "<username>/<image-name>:$CI_COMMIT_SHORT_SHA" -f
    - docker system prune -f
```

## Testing Node.js

```yaml
backend-server-image-test:
  image: <username>/<image-name>:$CI_COMMIT_SHORT_SHA
  stage: test
  script:
    - npm --prefix /usr/src/app test
    - exit
```

## Cleanup

```yaml
cleanup:
  stage: .post
  image: docker:19.03.1
  script:
    - docker login -u <username> -p <password>
    - docker image rm "<username>/<image-name>:$CI_COMMIT_SHORT_SHA" -f
    - docker system prune -f
```

## Image Delivery To Kubernetes (test deployment)

```yaml
backend-server-image-deliver-test-environment:
  environment:
    name: k8s-test
    url: 172.31.52.51:30012
  image: <username>/ubuntu-for-ssh:20200514-stable-1
  stage: deploy
  variables:
    CI_CONTAINER_IMAGE: <username>\/<image-name>:$CI_COMMIT_SHORT_SHA
  before_script:
    - echo "172.31.52.51 ecdsa-sha2-nistp256 <ssh-key>" > /root/.ssh/known_hosts
  script:
    - sshpass -p macroview ssh 172.31.52.51 << EOL
    - cd /root/billy/web-based-defacement-detection/deployment/test
    - sed -i -E "s/(image:\s).*/\1$CI_CONTAINER_IMAGE/" backend.yaml
    - kubectl apply -f backend.yaml
    - >
      if ! kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>; then
          kubectl rollout undo deployment <k8s-deployment-name> -n <k8s-namespace>
          kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>
          exit 1
      fi
    - EOL
  except:
    - master
```

## Image Delivery To Kubernetes (production deployment)

```yaml
backend-server-image-deliver-test-environment:
  environment:
    name: k8s-test
    url: 172.31.52.51:30012
  image: <username>/ubuntu-for-ssh:20200514-stable-1
  stage: deploy
  variables:
    CI_CONTAINER_IMAGE: <username>\/<image-name>:$CI_COMMIT_SHORT_SHA
  before_script:
    - echo "172.31.52.51 ecdsa-sha2-nistp256 <ssh-key>" > /root/.ssh/known_hosts
  script:
    - sshpass -p macroview ssh 172.31.52.51 << EOL
    - cd /root/billy/web-based-defacement-detection/deployment/test
    - sed -i -E "s/(image:\s).*/\1$CI_CONTAINER_IMAGE/" backend.yaml
    - kubectl apply -f backend.yaml
    - >
      if ! kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>; then
          kubectl rollout undo deployment <k8s-deployment-name> -n <k8s-namespace>
          kubectl rollout status deployment <k8s-deployment-name> -n <k8s-namespace>
          exit 1
      fi
    - EOL
```
