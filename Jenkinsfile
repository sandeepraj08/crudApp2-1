pipeline {
    agent any
    environment {
        DOCKER_IMAGE_NAME = "sandeep08/crudapp"
        CANARY_REPLICAS = 0
    }
    tools {
        maven 'maven'
    }
        stages {
            stage('Build') {
                steps {
		    
                    sh 'mvn -Dmaven.test.failure.ignore=true clean package'
                    //sh "'${mvnHome}/bin/mvn' -Dmaven.test.failure.ignore clean package"
                    
		}
            }
            stage ('Upload to Nexus') {
                steps {
                    nexusArtifactUploader artifacts: [[artifactId: 'crudApp', classifier: '', file: 'target/crudApp.war', type: 'war']], credentialsId: 'nexus', groupId: 'Central', nexusUrl: '$NEXUS_IP', nexusVersion: 'nexus2', protocol: 'http', repository: 'releases', version: '1.${BUILD_NUMBER}'
                }
            }
            stage ('Docker Build') {
                when {
                    branch 'master'
                }
                steps {
                    sh 'wget http://$NEXUS_IP/service/local/repositories/releases/content/Central/crudApp/1.${BUILD_NUMBER}/crudApp-1.${BUILD_NUMBER}.war -O crudApp.war'
                    script {
                        app = docker.build(DOCKER_IMAGE_NAME)
                    }
                    sh 'rm -rf crud*'
                }
            }
           stage ('Docker Push Image') {
                steps{
                    script {
                        docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                            app.push("${env.BUILD_NUMBER}")
                            app.push("latest")
                        }
                    }
                }
            }
            stage('CanaryDeploy') {
               when {
                    branch 'master'
                }
                environment {
                    CANARY_REPLICAS = 1
                }
                steps {
                    kubernetesDeploy(
                        kubeconfigId: 'kubeconfig',
                        configs: 'canary-kube.yml',
                        enableConfigSubstitution: true
                    )
                }
            }
            stage('SmokeTest') {
                when {
                    branch 'master'
                }
                steps {
                    script {
                        sleep (time: 25)
                        def response = httpRequest (
                            url: "http://$KUBE_MASTER_IP:30001/crudApp",
                            timeout: 30
                        )
                        if (response.status != 200) {
                            error("Smoke test against canary deployment failed.")
                        }
                    }
                }
            }
            stage ('Deploy To Production') {
                when {
                    branch 'master'
                }
                steps {
                    input 'Deploy to Production?'
                    milestone(1)
                    kubernetesDeploy(
                        kubeconfigId: 'kubeconfig',
                        configs: 'kube',
                        enableConfigSubstitution: true
                    )
                }
            }
        }
}
