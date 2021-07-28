# Setup CI with Jenkins and Gitlab
## 1. Workflow:
- When you commit to gitlab repository, Gilab send webhook to Jenkins and Jenkins will execute job.
- Jenkins will execute:
    - Checkout branch va pull code.
    - Build project (vd: Java, code the la build bang maven hoac gradle).
      - Maven: `mvn clean package`, gradle: `./gradlew clean build`
    - Build Docker image using Dockerfile
    - Publish Docker image to any Registry. Ex: Gitlab Registry
    - Remove docker image build in the previous step


## 2. Setup:
- Jenkin side:
  - Run Jenkins container.

    ```
    docker run -v /var/run/docker.sock:/var/run/docker.sock -v $(which docker):$(which docker) -v `pwd`/jenkins_data:/var/jenkins_home -p 8081:8080 --user 1000:998 --name jenkins-server -d jenkins/jenkins:jdk11

    #Explain:
      1. `-v $(which docker):$(which docker)`: dung de mount docker tu may host vao trong Jenkins container, de trong Jenkins container co the build image.

      2. `-v `pwd`/jenkins_data:/var/jenkins_home`: tao thu muc "jenkins_data" de mount data cua jenkins container, du dung trong truowng hop Jenkins container stop hoac bi xoa thi se khong bi mat du lieu.

      3. '-p 8081:8080': mapping port may host voi port cua container. Jenkins con chay cong 50000, neu muon chay Jenkins master-slave thi co the mapping port them "-p 50000:50000"
      
      4. '--user 1000:998': trong Jenkins container se su dung default user "jenkins" =>  se khong co quyen de build docker image, su dung lenh nay de mount user tu may host vao ben trong jenkins_container
      "1000": 1000 o day laf id cua user may host -> run command: "id" or "whoiam" de xem id cua user va group cua user dang join.
      "998": group docker o may host.
      5. "-d": la chay o che do detach.

      6. "jenkins/jenkins:jdk11": ten image cua jenkins(image nay su dung jdk:11), tuy thuoc vao code project su dung java8 hay java11 se su dung image cho hop ly.

    ```
  - Login Jenkin and install plugin suggest:
    - Attach to Jenkins container and run

      ```
      sudo cat /var/lib/jenkins/secrets/initialAdminPassword
      ```
    - After install all plugin jenkins suggest -> create job with type: "Pipeline"
    - Create "Gitlab Credential":
      - Click "Manage Jenkins" -> click "Manage Credentials" -> Dien user/pass Gitlab.
    - Create Jenkin job with type "Pipiline":
      - Click to Jenkins Job -> click "Configure" -> scroll to "Pipeline" -> select "Pipeline script from SCM" -> pass gitlab repository url and credential created in the previous step -> click "apply" and "save".

- Gitlab project side:
  - Create Jenkinsfile in root project.
    ```
    pipeline{
      environment {
        def REGISTRY_URL = "GITLAB_REGISTRY OR DOCKER HUB,..."
        def IMAGE_NAME = "${REGISTRY_URL}/<project-path-to registry>/images/<name images>"
        def IMAGE_TAG = "${GIT_BRANCH.tokenize('/').pop()}-${GIT_COMMIT.substring(0,8)}"
        def DOCKER_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
      }

     agent any

      stages{
          stage('Checkout branch') {
            steps {
                git branch: '<branch muon pull code>', credentialsId: '<credential-id-create on jenkins >', url: '<url_git>'
            }
          }
          stage('Gradle build') {
            steps {
                sh """
                    chmod +x gradlew  //cap quyen cho lenh gradlew
                    ./gradlew --version
                    echo $JAVA_HOME
                    ./gradlew clean build // clean and build project -> jar file in foler "build/libs/."
                """
              }
          }
          stage('Build docker image') {
            steps {
                sh """
                    echo DOCKER_IMAGE: ${DOCKER_IMAGE}
                    docker build -t ${DOCKER_IMAGE} .
                """
              }
          }
          stage('Publish docker image') {
              steps {
                withCredentials([usernamePassword(credentialsId: '<credential-gitlab-create-jenkins>', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh 'echo $DOCKER_PASSWORD | docker login ${REGISTRY_URL} --username $DOCKER_USERNAME --password-stdin'
                    sh "docker push ${DOCKER_IMAGE}"
                }

                //clean docker image
                sh 'docker rmi ${DOCKER_IMAGE}'
              }
          }
      }
      post{
          success{
            echo "========pipeline executed successfully ========"
            updateGitlabCommitStatus name: 'build', state: 'success'
          }
          failure{
            echo "========pipeline execution failed========"
            updateGitlabCommitStatus name: 'build', state: 'failed'
          }
        }
      }


    ```

## 3. Setup webhook:
- Document ref: https://news.cloud365.vn/ci-cd-phan-3-huong-dan-tich-hop-jenkins-va-gitlab/