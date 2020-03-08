def jenkinsSlaveImage = "cne-repos1.us.oracle.com:7744/apps/cgbu/analytics/feature/use_java_11_instead_of_8/img-jenkins-slave:19.9.0.11"
def component = "sjs"
def ms_name = "sjs"

/*library identifier: 'common-utils@develop', retriever: modernSCM(
        [$class       : 'GitSCMSource',
         remote       : 'ssh://git@cloudlab.us.oracle.com:2222/OCAS/common-utils.git',
         credentialsId: 'Jenkins'])

library identifier: getLibraryName(), retriever: modernSCM(
        [$class       : 'GitSCMSource',
         remote       : 'ssh://git@cloudlab.us.oracle.com:2222/OCAS/common-jenkins.git',
         credentialsId: 'Jenkins'])*/

pipeline {
    agent any

    stages {


        stage('Compile') {
            steps {
                echo "Compiling..."
                sh "sbt compile"
            }
        }

        stage('Test') {
            steps {
                echo "Testing..."
                sh "/usr/local/bin/sbt test"
            }
        }

        stage('Package') {
            steps {
                echo "Packaging..."
                sh "sbt package"
            }
        }

        stage("Build"){
                    steps{
                        sh "sbt run"
                    }
                }

    }
}