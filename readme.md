# Tooling

Steps were performed on MacOs Monterey v12.7.1 with an Intel chip.

# Steps

1. Install Docker by visiting [https://docs.docker.com/desktop/install/mac-install/](https://docs.docker.com/desktop/install/mac-install/)
2. Select [Docker Desktop for Mac with Intel chip](https://desktop.docker.com/mac/main/amd64/Docker.dmg?utm_source=docker&utm_medium=webreferral&utm_campaign=docs-driven-download-mac-amd64&_gl=1*1w6mch1*_ga*MjQ3NDk1NTQwLjE3MDAyODIxODA.*_ga_XJWPQMJYHQ*MTcwMDY4ODQ3My43LjEuMTcwMDY4ODQ3Ni41Ny4wLjA.)
3. Wait for the file to complete downloading
4. OS will present a popup prompting you to drag the Docker download into the Applications folder. Do so.
5. Double-click Docker in the Applications folder to start the docker engine
6. The OS will prompt “Docker is an application dowloaded from the internet, are you sure you want to run it?”. Click “Yes”.
7. Set up the containers by opening a terminal and running `sh provision.sh` in the **scripts** folder containing `provision.sh` (this script will accomplish steps #8 - #16). For reference you can inspect `provision.sh` in the scripts section of this document.
8. Creates a docker network `petclinic_network` (which the containers for our pipeline will belong to)
9. Creates volumes `sonarqube_data` and `jenkins_data` to persist data for each of the containers
10. Pulls the official docker `sonarqube` image
11. Runs the`sonarqube` image as a container in detached mode, with the port `9000` mapped to the localhost `9000` port, with the `sonarqube_data` volume attached, and as part of the `petclinic_network` under the name `sonarqube`
12. Makes a directory`petclinic` to build the image for the Jenkins container
13. Navigates into `petclinic`
14. Creates a `Dockerfile` with commands to do the following: Pull the official Docker `ubuntu` image, install Jenkins and its dependencies, and other tools leveraged in the pipeline and container: `maven`, `git`, `fontconfig`,`openjdk-17-jre`, and `curl`, expose and runs Jenkins on port `8081`
15. Builds the image with the name `ubuntu-jenkins`
16. Runs the image as a container named `jenkins` on the `petclinic_network`, with the volume `jenkins_data`, and container ports `8082`, `8081,` and `50000` mapped to their `localhost` counterparts
17. Now that the `sonarqube` app is running navigate to `localhost:9000` and see the UI
18. Login with the default creds “admin” “admin”
19. Create new creds and finish logging in
20. On the main dashboard select “Create Local Project”
21. Create a project with the name and key `spring-petclinic` and “main branch name” set to`main`
22. Choose the baseline global settings for the project’s code definitions
23. Select “Create Project” to finish project creation
24. Navigate to “My account” > “Security”
25. Generate a project analysis level token called `spring-petclinic token` for the `spring-petclinic` and project, set to expire in 30 days
26. Copy the token for later plugin configuration inside Jenkins
27. Navigate back to the terminal and copy the admin token from the Jenkins container run log
28. Navigate to `localhost:8081` to find the Jenkins web app running.
29. Paste the admin token to unlock Jenkins
30. Elect to Install the suggested plugins and wait for them to install
31. Set up custom credentials
32. Click “Save and Finish”
33. Click “Start using Jenkins”
34. Navigate to “Dashboard” > “Manage Jenkins” > “Plugins” > “Available Plugins”
35. Search for the plugin `SonarQube Scanner`
36. Select `SonarQube Scanner` and click install
37. At the bottom of the install screen check “Restart Jenkins when installation is complete and no jobs are running”
38. Repeat plugin installation steps for the `Blue Ocean` plugin
39. Navigate to “Manage Jenkins” > “System” > “SonarQube servers”
40. Click “Add SonarQube”
41. Enter SonarQube installation name as `spring-petclinic`
42. Enter Server URL as [http://sonarqube:9000](http://sonarqube:9000) (`sonarqube` is the `petclinic_network` name for the `sonarqube` container, so this URL will point to the `sonarqube` container's IP, and `9000` the port the SonarQube application is running on within its container)
43. Save (you will need to re-navigate to this screen to add “Server authentication token”, it is a weird bug)
44. Re-navigate to “Manage Jenkins” > “System” > “SonarQube servers”
45. In the existing SonarQube installation under “Server authentication token”, click “Add” > “Jenkins” to add a new Jenkins credential
46. Create a new Jenkins credentials of “domain” `Global Credentials`, kind `Secret Text`, and description `spring-petclinic token`. For the “secret” paste the token copied in step #26
47. Set the `spring-petclinic token` credential as the “Server authentication” token for the SonarQube installation
48. Now start to create the pipeline by navigating to the main dashboard, selecting “+ New item”
49. Enter the name `petclinic-pipeline`
50. Choose type “Pipeline”
51. Click “OK”
52. You are navigated to the “Configure” page
53. Under “Configure” > “Pipeline” > “Script” enter **pipeline-script.txt** found in the **scripts** folder which accomplishes the following steps #54 - #56. For reference you can inspect **pipeline-script.txt** in the scripts section of this document.
54. Add “Build Stage” with the below code, which removes code from previous pipeline runs, clones the `spring-petclinic` repo from Github, and builds the app:
    ```java
      stage('Build') {
            sh "rm -rf ${workspace}/spring-petclinic && 
            git clone 'https://github.com/spring-projects/spring-petclinic'"
            sh "cd ${workspace}/spring-petclinic && ./mvnw package"
        }
    ```
55. Add “SonarQube stage” with the below code which uses Maven to run the SonarQube scanner
    ```java
      stage('Sonar qube') {
            withSonarQubeEnv() {
                sh "mvn -f ${workspace}/spring-petclinic/pom.xml 
                clean verify sonar:sonar -Dsonar.projectKey='spring-petclinic'"            
            }
        }
    ```
56. Add Deploy stage with the below code, which runs the app on `localhost:8082`
    ```java
    stage('Deploy') {
      sh 'JENKINS_NODE_COOKIE=dontKillMe nohup java -Dserver.port=8082 -jar 
      /root/.jenkins/workspace/petclinic-pipeline/spring-petclinic/target/*.jar &'
    }
    ```
57. You are navigated back to `petclinic-pipeline` screen
58. Click “Build now”
59. Watch the pipeline build and wait for it to finish successfully
60. Open `localhost:8082` to view the spring-petclinic app landing page

# Scripts

Please see **scripts** folder for script files.

### provision.sh

```java
docker network create petclinic_network
docker volume create sonarqube_data
docker volume create jenkins_data
docker pull sonarqube
docker run -d  --name sonarqube -p 9000:9000 -v sonarqube_data:/opt/sonarqube/data --network petclinic_network sonarqube
mkdir petclinic
cd petclinic
echo '
FROM ubuntu:latest

# Install required packages
RUN apt-get upgrade &&  apt-get update && apt-get install -y maven git fontconfig openjdk-17-jre curl

# Add Jenkins GPG key
RUN curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | tee   /usr/share/keyrings/docker-archive-keyring.asc > /dev/null

RUN echo deb [signed-by=/usr/share/keyrings/docker-archive-keyring.asc]   https://pkg.jenkins.io/debian binary/ | tee   /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package lists and install Jenkins
RUN apt-get update && apt-get install -y jenkins

# Expose the Jenkins port
EXPOSE 8081

# Start Jenkins
CMD ["java", "-jar", "/usr/share/java/jenkins.war", "--httpPort=8081"]

' > Dockerfile 
docker build -t ubuntu-jenkins . --no-cache
docker run --name jenkins -p 8082:8082 -p 8081:8081 -p 50000:50000 -v jenkins_data:/root/.jenkins --network petclinic_network ubuntu-jenkins

```

### pipeline-script.txt

```java
node {
    stage('Build') {
        sh "rm -rf ${workspace}/spring-petclinic &&  git clone 'https://github.com/spring-projects/spring-petclinic'"
        sh "cd ${workspace}/spring-petclinic && ./mvnw package"
    }
    stage('Sonar qube') {
        withSonarQubeEnv() {
            sh "mvn -f ${workspace}/spring-petclinic/pom.xml clean verify sonar:sonar -Dsonar.projectKey='spring-petclinic'"            
        }
    }
    stage('Deploy') {
            sh 'JENKINS_NODE_COOKIE=dontKillMe nohup java -Dserver.port=8082 -jar /root/.jenkins/workspace/petclinic-pipeline/spring-petclinic/target/*.jar &'
    }
}
```

# References

1. [https://docs.docker.com/reference/](https://docs.docker.com/reference/)
2. [https://hub.docker.com/_/sonarqube](https://hub.docker.com/_/sonarqube)
3. [https://docs.sonarsource.com/sonarqube/latest/](https://docs.sonarsource.com/sonarqube/latest/)


