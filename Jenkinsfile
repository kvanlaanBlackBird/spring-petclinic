node {
    checkout scm
      def mvnHome
    stage('Build') {
        // Run the maven build
                sh "rm -rf ${workspace}/spring-petclinic &&  git clone 'https://github.com/kvanlaanBlackBird/spring-petclinic'"
                sh "cd ${workspace}/spring-petclinic && ./mvnw package"
    }
    stage('Sonar qube') {
             withMaven {
                    withSonarQubeEnv() {
                        sh "mvn clean verify sonar:sonar"               
                    } 
             }
    }
    stage('Deploy') {
             sh 'JENKINS_NODE_COOKIE=dontKillMe nohup java -Dserver.port=8082 -jar /root/.jenkins/workspace/petclinic/spring-petclinic/target/*.jar &'
    }
}
