</details>

******

<details>
<summary>Exercise 1: Dockerize your NodeJS App </summary>
 <br />

**Create Dockerfile in the root folder of the project**
```sh
FROM node:13-alpine

RUN mkdir -p /usr/app
COPY app/* /usr/app/

WORKDIR /usr/app
EXPOSE 3000

RUN npm install
CMD ["node", "server.js"]

```

</details>

******

<details>
<summary>Exercise 2: Create a full pipeline for your NodeJS App </summary>
 <br />

**Create Jenkins Credentials**
- Create usernamePassword credentials for docker registry called `docker-credentials`
- Create usernamePassword credentials for git repositoriy called `gitlab-credentials`

**Configure Node Tool in Jenkins Configuration**
- Name should be `node`, because that's how it's referenced in the below Jenkinsfile in `tools` block

**Install plugin**
- Install `Pipeline Utility Steps` plugin. This contains readJSON function, that we will use to read the version from package.json 

**Jenkinsfile**

```sh
pipeline {
    agent any
    tools {
        nodejs "node"
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    # enter app directory, because that's where package.json is located
                    dir("app") {
                        # update application version in the package.json file with one of these release types: patch, minor or major
                        # this will commit the version update
                        npm version minor

                        # read the updated version from the package.json file
                        def packageJson = readJSON file: 'package.json'
                        def version = packageJson.version

                        # set the new version as part of IMAGE_NAME
                        env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                    }

                    # alternative solution without Pipeline Utility Steps plugin: 
                    # def version = sh (returnStdout: true, script: "grep 'version' package.json | cut -d '\"' -f4 | tr '\\n' '\\0'")
                    # env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('Run tests') {
            steps {
               script {
                    # enter app directory, because that's where package.json and tests are located
                    dir("app") {
                        # install all dependencies needed for running tests
                        sh "npm install"
                        sh "npm run test"
                    } 
               }
            }
        }
        stage('Build and Push docker image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'USER', passwordVariable: 'PWD')]){
                    sh "docker build -t docker-hub-id/myapp:${IMAGE_NAME} ."
                    sh "echo ${PWD} | docker login -u ${USER} --password-stdin"
                    sh "docker push docker-hub-id/myapp:${IMAGE_NAME}"
                }
            }
        }
        stage('commit version update') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'gitlab-credentials', usernameVariable: 'USER', passwordVariable: 'PWD')]) {
                        # git config here for the first time run
                        sh 'git config --global user.email "jenkins@example.com"'
                        sh 'git config --global user.name "jenkins"'

                        sh "git remote set-url origin https://${USER}:${PWD}@gitlab.com/devops-bootcamp3/node-project.git"
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:jenkins-jobs'
                    }
                }
            }
        }
    }
}

```



</details>

******

<details>
<summary>Exercise 3: Manually deploy new Docker Image on server </summary>
 <br />

**steps:**
```sh
# ssh into your droplet server
ssh -i ~/id_rsa root@{server-ip-address}

# login to your docker hub registry
docker login

# pull and run the new docker image from registry
docker run -p 3000:3000 {docker-hub-id}/myapp:{image-name}

```

</details>

******

<details>
<summary>Exercise 4: Extract into Jenkins Shared Library </summary>
 <br />

**steps:**

- Create a separate git repo for Jenkins Shared Library 
- Extract code withing the script blocks from `increment version`, `Run tests`, `Build and Push docker image` and `commit version update` to Jenkins Shared Library
- Configure your Jenkinsfile to use the Jenkins Shared Library project

Reference the demo videos in the module for these steps. All of these is explained in detail there.
To validate that your code is correct, execute the pipeline when done. If you get the same result as before and you have a new image in the reigstry at the end of the pipeline execution, then you have successfully completed the exercise.  

</details>

******

