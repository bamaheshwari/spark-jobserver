def jenkinsSlaveImage = "cne-repos1.us.oracle.com:7744/apps/cgbu/analytics/feature/use_java_11_instead_of_8/img-jenkins-slave:19.9.0.11"
def component = "sjs"
def ms_name = "sjs"

library identifier: 'common-utils@develop', retriever: modernSCM(
        [$class       : 'GitSCMSource',
         remote       : 'ssh://git@cloudlab.us.oracle.com:2222/OCAS/common-utils.git',
         credentialsId: 'Jenkins'])

library identifier: getLibraryName(), retriever: modernSCM(
        [$class       : 'GitSCMSource',
         remote       : 'ssh://git@cloudlab.us.oracle.com:2222/OCAS/common-jenkins.git',
         credentialsId: 'Jenkins'])

pipeline {
    agent {
        kubernetes {
            label "ocas-build-${UUID.randomUUID().toString()}"
            inheritFrom 'dind'

            containerTemplate {
                name 'ocas-build'
                image "$jenkinsSlaveImage"
                ttyEnabled true
                alwaysPullImage true
            }
        }
    }
    parameters {
        string(name: 'project_config', defaultValue: 'devcorp', description: 'Project related configuration')
        string(name: 'environment_config', defaultValue: 'development', description: 'Environment related configuration')
        string(name: 'deploy_artifacts', defaultValue: 'false', description: 'Condition to deploy artifacts on CNE or not')
    }
    options {
        timestamps()
        disableConcurrentBuilds()
    }
    environment {
        OCAS_MASTER_ACCESS_TOKEN = credentials("OCAS_MASTER_ACCESS_TOKEN")
        DEPLOYER = credentials("ocas-deployer-creds")
        OCAS_DEPLOYER_USER = "${env.DEPLOYER_USR}"
        OCAS_DEPLOYER_PASSWORD = "${env.DEPLOYER_PSW}"
        ARTIFACT_REPO = credentials("ocas-development-artifact-repo")
        OCAS_DEVELOPMENT_ARTIFACT_REPO_USER = "${env.ARTIFACT_REPO_USR}"
        OCAS_DEVELOPMENT_ARTIFACT_REPO_PASS = "${env.ARTIFACT_REPO_PSW}"
        DOCKER_HOST = "tcp://localhost:2375"
        VERSION_BRANCH = getVersionBranch()
        VERSION = setVersion()
    }
    stages {

        stage('Build') {
            steps {
                script {
                    sh sbt run
                }
            }
        }
      /*  stage('Checkstyle and PMD') {
            steps {
                script {
                    sh "./gradlew check -x test"
                }
            }
        }
        stage('Unit Test') {
            steps {
                script {
                    sh "./gradlew test"
                }
            }
        }

        stage('Test Coverage') {
            steps {
                script {
                    sh "./gradlew jacocoTestCoverageVerification"
                    sh "./gradlew jacocoTestReport"
                }
            }
        }*/
        /*stage('Publish Test Results') {
            steps {
                script {
                    sh "./gradlew jacocoTestReport sonarqube"
                }
            }
        }*/
        stage('Docker Lint Check') {
            steps {
                script {
                    withProjectConfig("${params.project_config}") {
                        dockerLint()
                    }
                }
            }
        }
        stage('YAML Lint Check') {
            steps {
                script {
                    withProjectConfig("${params.project_config}") {
                        yamlLint()
                    }
                }
            }
        }
        stage('Build & Publish Docker Image') {
            steps {
                script {
                    withProjectConfig("${params.project_config}") {
                        buildImage(component, "base-java-11")
                    }
                }
            }
        }
        stage('Upload Artifacts') {
            steps {
                script {
                    withProjectConfig("${params.project_config}") {
                        uploadArtifacts(component)
                    }
                }
            }
        }
        stage('Scan Artifacts') {
            when {
                anyOf {
                    branch "${VERSION_BRANCH}"
                    branch "develop"
                    branch "release"
                }
            }
            steps {
                script {
                    withProjectConfig("${params.project_config}") {
                        withEnvConfig("${params.environment_config}") {
                            performScans(component, "all")
                            //performScans(component, "fortify")
                        }
                    }
                }
            }
        }
        stage('Sync Artifacts') {
            steps {
                script {
                    withProjectConfig("${params.project_config}") {
                        withEnvConfig("${params.environment_config}") {
                            syncArtifacts(component)
                        }
                    }
                }
            }
        }
        stage('Deploy Artifacts') {
            when{
                expression{
                    return "${params.deploy_artifacts}".toBoolean()
                }
            }
            steps {
                script {
                    withProjectConfig("${params.project_config}") {
                        withEnvConfig("${params.environment_config}") {
                            deployArtifacts(component)
                        }
                    }
                }
            }
        }
        stage('Integration Tests') {
            when {
                allOf {
                    not
                    {
                        branch 'release'
                    }
                    expression {
                        return !checkUpstreamProject("OCAS_service-onboarding")
                    }
                }
            }
            steps {
                script {
                    def params = [[$class: 'StringParameterValue', name: 'project_config', value: params.project_config],
                                  [$class: 'StringParameterValue', name: 'environment_config', value: params.environment_config]]

                    // To test changes in feature branch of integration change the name "develop" in below lines of code. Please before merging change it to develop.

                    build job: "OCAS_integration/${env.BRANCH_NAME}", parameters: params, propagate: true, wait: true
                }
            }
        }
        stage('Version Update') {
            when {
               anyOf {
                    branch "${VERSION_BRANCH}"
                    branch "develop"
                    branch "release"
               }
            }
            steps {
                script {
                   sh''' git clone https://Jenkins:szc_p7Zb1FEkwk6zhR1-@cloudlab.us.oracle.com/OCAS/config-onboarding.git --branch=${BRANCH_NAME} '''
                   versionUpdate(ms_name)
            }
        }
    }
        stage('Tag Release') {
            when {
                anyOf {
                    branch "release"
                }
            }
            steps {
                script {
                    releaseTag()
                }
            }
        }
    }
    post {
        failure {
            script {
                currentBuild.result = 'FAILURE'
            }
        }
        always {
            archiveArtifacts artifacts: 'build/reports/', fingerprint:true
            jacoco 'build/reports/jacoco/test/html/*.html'
            script {
                sendEmails()
            }
        }
    }
}
