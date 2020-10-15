# Building DevSecOps into the pipeline using Aqua Security

The ***Shift Left*** approach to DevOps Security is becoming increasingly popular, which 
means that security must be built into the process and pipeline. One of the biggest 
problems with shortened pipelines is that they often leave little room for proper security 
checks. Due to this, another approach called ***deploy changes as quickly as possible*** was 
introduced, which is key to the success of DevOps.

In this section, we will cover automating vulnerability checks in container images using 
Aqua Security to reduce the application attack surface.

## Getting ready
Make sure you have an existing CI/CD pipeline configured using your preferred CI/CD 
tool. If not, follow the instructions in Chapter 3 , Building CI/CD Pipelines, to configure 
GitLab or CircleCI.

Clone the k8sdevopscookbook/src repository to your workstation to use the manifest 
files in the chapter9 directory, as follows:
```
$ git clone https://github.com/k8sdevopscookbook/src.git
$ cd src/chapter9
```
Make sure you have a Kubernetes cluster ready and kubectl configured to manage the cluster resources.

## How to do it...
This section will show you how to integrate Aqua with your CI/CD platform. This section is 
further divided into the following subsections to make this process easier:

* Scanning images using Aqua Security Trivy
* Building vulnerability scanning into GitLab
* Building vulnerability scanning into CircleCI

## Scanning images using Trivy
Trivy is an open source container scanning tool that's used to identify container 
vulnerabilities. It is one of the simplest and most accurate scanning tools in the market. In 
this recipe, we will learn how to install and scan container images using Trivy.

Let's perform the following steps to run Trivy:

1. Get the latest Trivy release number and keep it in a variable:
```
$ VERSION=$(curl --silent "https://api.github.com/repos/aquasecurity/trivy/releases/latest" |\
grep '"tag_name":' | \
sed -E 's/.*"v([^"]+)".*/\1/')
```
Download and install the trivy command-line interface:
```
$ curl --silent --location "https://github.com/aquasecurity/trivy/releases/download/v${VERSION}/trivy_${VERSION}_Linux-64bit.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/trivy /usr/local/bin
```
3. Verify that trivy is functional by running the following command. It will return its current version:
```
$ trivy --version
trivy version 0.12.0
```
4. Execute trivy checks by replacing the container image name with your target 
image. In our example, we scanned the postgres:12.0 image from the Docker 
Hub repository:
```
$ trivy postgres:12.0
2020-10-15T19:47:48.540+0800	INFO	Need to update DB
2020-10-15T19:47:48.540+0800	INFO	Downloading DB...
2020-10-15T19:49:51.119+0800	INFO	Detecting Debian vulnerabilities...

postgres:12.0 (debian 10.1)
===========================
Total: 281 (UNKNOWN: 2, LOW: 181, MEDIUM: 92, HIGH: 6, CRITICAL: 0)
...
```
5. The test summary will show the number of vulnerabilities that have been 
detected and will include a detailed list of vulnerabilities, along with their IDs 
and an explanation of each of them:
```
+----------------------+---------------------+----------+------------------------+------------------------+-----------------------------------------+
|       LIBRARY        |  VULNERABILITY ID   | SEVERITY |   INSTALLED VERSION    |     FIXED VERSION      |                  TITLE                  |
+----------------------+---------------------+----------+------------------------+------------------------+-----------------------------------------+
| apt                  | CVE-2020-3810       | MEDIUM   | 1.8.2                  | 1.8.2.1                | Missing input validation in             |
|                      |                     |          |                        |                        | the ar/tar implementations of           |
|                      |                     |          |                        |                        | APT before version 2.1.2...             |
+                      +---------------------+----------+                        +------------------------+-----------------------------------------+
|                      | CVE-2011-3374       | LOW      |                        |                        | It was found that apt-key               |
|                      |                     |          |                        |                        | in apt, all versions, do not            |
|                      |                     |          |                        |                        | correctly...                            |
+----------------------+---------------------+          +------------------------+------------------------+-----------------------------------------+
| bash                 | CVE-2019-18276      |          | 5.0-4                  |                        | bash: when effective UID is             |
|                      |                     |          |                        |                        | not equal to its real UID               |
|                      |                     |          |                        |                        | the...                                  |
+                      +---------------------+          +                        +------------------------+-----------------------------------------+
|                      | TEMP-0841856-B18BAF |          |                        |                        |                                         |
+----------------------+---------------------+          +------------------------+------------------------+-----------------------------------------+
| coreutils            | CVE-2016-2781       |          | 8.30-3                 |                        | coreutils: Non-privileged               |
|                      |                     |          |                        |                        | session can escape to the               |
|                      |                     |          |                        |                        | parent session in chroot                |
+                      +---------------------+          +                        +------------------------+-----------------------------------------+
|                      | CVE-2017-18018      |          |                        |                        | coreutils: race condition               |
|                      |                     |          |                        |                        | vulnerability in chown and              |
|                      |                     |          |                        |                        | chgrp                                   |
+----------------------+---------------------+          +------------------------+------------------------+-----------------------------------------+
| dirmngr              | CVE-2019-14855      |          | 2.2.12-1+deb10u1       |                        | gnupg2: OpenPGP Key                     |
|                      |                     |          |                        |                        | Certification Forgeries with            |
|                      |                     |          |                        |                        | SHA-1                                   |
+----------------------+---------------------+----------+------------------------+------------------------+-----------------------------------------+
| e2fsprogs            | CVE-2019-5188       | MEDIUM   | 1.44.5-1+deb10u2       | 1.44.5-1+deb10u3       | e2fsprogs: Out-of-bounds write          |
|                      |                     |          |                        |                        | in e2fsck/rehash.c                      |
+----------------------+---------------------+          +------------------------+------------------------+-----------------------------------------+
| exim4-base           | CVE-2020-12783      |          | 4.92-8+deb10u3         | 4.92-8+deb10u4         | exim: out-of-bounds read in             |
|                      |                     |          |                        |                        | the SPA authenticator can lead          |
|                      |                     |          |                        |                        | to SPA/NTLM authentication...           |
+----------------------+                     +          +                        +                        +                                         +
| exim4-config         |                     |          |                        |                        |                                         |

...
```
With that, you've learned how to quickly scan your container images. Trivy supports a 
variety of container base images (CentOS, Ubuntu, Alpine, Distorless, and so on) and 
natively supports container registries such as Docker Hub, Amazon ECR, and Google 
Container Registry GCR. Trivy is completely suitable for CI. In the next two recipes, you 
will learn how you can add Trivy into CI pipelines.

## Building vulnerability scanning into GitLab
With GitLab Auto DevOps, the container scanning job uses CoreOS Clair to analyze Docker 
images for vulnerabilities. However, it is not a complete database of all security issues for 
Alpine-based images. Aqua Trivy has nearly double the number of vulnerabilities and is 
more suitable for CI. For a detailed comparison, please refer to the Trivy Comparison link in 
the See also section. This recipe will take you through adding a test stage to a GitLab CI 
pipeline.

Let's perform the following steps to add Trivy vulnerability checks in GitLab:

1. Edit the CI/CD pipeline configuration .gitlab-ci.yml file in your project:
```
$ vim .gitlab-ci.yml
```
2. Add a new stage to your pipeline and define the stage. You can find an example in the src/chapter9/devsecops directory. In our example, we're using 
the vulTest stage name:
```
stages:
  - build
  - vulTest
  - staging
  - production
#Add the Step 3 here
```

3. Add the new stage, that is, vulTest . When you define a new stage, you specify a 
stage name parent key. In our example, the parent key is trivy . The commands 
in the before_script section will download the trivy binaries:
```
trivy:
  stage: vulTest
  image: docker:stable-git
  before_script:
    - docker build -t trivy-ci-test:${CI_COMMIT_REF_NAME} .
    - export VERSION=$(curl --silent "https://api.github.com/repos/aquasecurity/trivy/releases/latest" |grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
    - wget https://github.com/aquasecurity/trivy/releases/download/v${VERSION}/trivy_${VERSION}_Linux-64bit.tar.gz
    - tar zxvf trivy_${VERSION}_Linux-64bit.tar.gz 
  variables:
    DOCKER_DRIVER: overlay2
  allow_failure: true
  services:
    - docker:stable-dind
#Add the Step 4 here
```
4. Finally, review and add the Trivy scan script and complete the vulTest stage. 
The following script will return --exit-code 1 for the critical severity 
vulnerabilities, as shown here:
```
script:
- ./trivy --exit-code 0 --severity HIGH --no-progress --auto-refresh trivy-ci-test:${CI_COMMIT_REF_NAME}
- ./trivy --exit-code 1 --severity CRITICAL --no-progress --auto-refresh trivy-ci-test:${CI_COMMIT_REF_NAME}
  cache:
    directories:
      - $HOME/.cache/trivy
```
Now, you can run your pipeline and the new stage will be included in your pipeline. The 
pipeline will fail if a critical vulnerability is detected. If you don't want the stage to fail your 
pipeline, you can also specify --exit-code 0 for critical vulnerabilities.

## Building vulnerability scanning into CircleCI
CircleCI uses Orbs to wrap predefined examples to speed up your project configurations. 
Currently, Trivy doesn't have a CircleCI Orb, but it is still easy to configure Trivy with 
CircleCI. This recipe will take you through adding a test stage to the CircleCI pipeline.

Let's perform the following steps to add Trivy vulnerability checks in CircleCI: 
1. Edit the CircleCI configuration file located in our project repository in .circleci/config.yml . You can find our example in the src/chapter9/devsecops directory:
```
$ vim .circleci/config.yml
```
2. Start by adding the job and the image. In this recipe, the job name is build :
```
jobs:
  build:
    docker:
      - image: docker:18.09-git
#Add the Step 3 here
```
3. Start adding the steps to build your image. The checkout step will checkout the project from its code repository. Since our job will require docker commands, add setup_remote_docker . When this step is executed, a remote environment will be created and your current primary container will be configured appropriately:
```
steps:
  - checkout
  - setup_remote_docker
  - restore_cache:
      key: vulnerability-db
  - run:
      name: Build image
      command: docker build -t trivy-ci-test:${CIRCLE_SHA1} .
#Add the Step 4 here
```
4. Add the necessary step to install Trivy:
```
- run:
name: Install trivy
command: |
  apk add --update curl
  VERSION=$(curl --silent "https://api.github.com/repos/aquasecurity/trivy/releases/latest" |\
            grep '"tag_name":' | \
            sed -E 's/.*"v([^"]+)".*/\1/') 
  wget https://github.com/aquasecurity/trivy/releases/download/v${VERSION}/trivy_${VERSION}_Linux-64bit.tar.gz
  tar zxvf trivy_${VERSION}_Linux-64bit.tar.gz
  mv trivy /usr/local/bin
#Add the Step 5 here
```
5. Add the step that will scan a local image with Trivy. Modify the trivy 
parameters and preferred exit codes as needed. Here, trivy only checks for 
critical vulnerabilities ( --severity CRITICAL ) and fails if a vulnerability is 
found ( --exit-code 1 ). It suppresses the progress bar ( --no-progress ) and 
refreshes the database automatically when updating its version ( --auto-refresh ):
```
- run:
    name: Scan the local image with trivy
    command: trivy --exit-code 1 --severity CRITICAL --no-progress --auto-refresh trivy-ci-test:${CIRCLE_SHA1}
- save_cache:
    key: vulnerability-db
    paths:
      - $HOME/.cache/trivy
#Add the Step 6 here
```    
6. Finally, update the workflows to trigger the vulnerability scan:
```
workflows:
  version: 2
  release:
  jobs:
  - build
```
Now, you can run your pipeline in CircleCI and the new stage will be included in your pipeline.
## See also
* Aqua Security Trivy Comparison: https://github.com/​ aquasecurity/trivy#comparison-​ with-​ other-​ scanners
* Aqua Security Trivy CI examples: https:/​ / ​ github.​ com/​ aquasecurity/trivy#comparison-​ with-​ other-​ scanners
* Aqua Security Trivy Alternatives for image vulnerability testing:
    * Aqua Security Microscanner: https:/​ / ​ github.​ com/​ aquasecurity/microscanner
    * Clair: https:/​ / ​ github.​ com/​ coreos/​ clair
    * Docker Hub: https:/​ / ​ beta.​ docs.​ docker.​ com/​ v17.​ 12/​ docker-cloud/​ builds/​ image-​ scan/​
    * GCR: https:/​ / ​ cloud.​ google.​ com/​ container-​ registry/​ docs/container-analysis
    * Layered Insight: https://layeredinsight.com/
    * NeuVector: https:/​ / ​ neuvector.​ com/​ vulnerability-​ scanning/​
    * Sysdig Secure: https:/​ / ​ sysdig.​ com/​ products/​ secure/​
    * Quay: https://coreos.com/​ quay-​ enterprise/​ docs/latest/security-​ scanning.html
    * Twistlock: https://www.​ twistlock.​ com/​ platform/vulnerability-​ management-​ tools/​
