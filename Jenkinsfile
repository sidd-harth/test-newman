pipeline {
    agent any

    tools {
        maven 'M386'
    }

    environment {
        POM_VERSION          = sh(script: "mvn help:evaluate -Dexpression='project.version' -q -DforceStdout", returnStdout: true).trim()
        POM_MAJOR_VERSION    = sh(script: "mvn help:evaluate -Dexpression='project.version' -q -DforceStdout |  cut -d. -f1", returnStdout: true).trim()
        POM_MINOR_VERSION    = sh(script: "mvn help:evaluate -Dexpression='project.version' -q -DforceStdout |  cut -d. -f2", returnStdout: true).trim()
        POM_PATCH_VERSION    = sh(script: "mvn help:evaluate -Dexpression='project.version' -q -DforceStdout |  cut -d. -f3 | cut -d- -f1", returnStdout: true).trim()
        POM_MAVEN_QUALIFIER  = sh(script: "mvn help:evaluate -Dexpression='project.version' -q -DforceStdout | cut -d- -f2", returnStdout: true).trim()
        POM_SCM_URL          = sh(script: "mvn help:evaluate -Dexpression='project.scm.url' -q -DforceStdout", returnStdout: true).trim()
        MULE_RUNTIME_VERSION = sh(script: "mvn help:evaluate -Dexpression='app.runtime' -q -DforceStdout", returnStdout: true).trim()
        ARTIFACT_ID          = sh(script: "mvn help:evaluate -Dexpression='project.artifactId' -q -DforceStdout", returnStdout: true).trim()
        ANYPOINT_PLATFORM_CREDENTIALS = credentials('ANYPOINT_PLATFORM_CREDENTIALS')
    }

    parameters {
        //string( name: 'BRANCH', description: 'Name of the Release Branch to be built? (e.g. release/1.1.0-1)' )
        choice( name: 'ENVIRONMENT', choices: "qa\nprod", description: 'Environment where Mule Application will be deployed' )
        string( name: 'API_VERSION', defaultValue: 'v1', description: 'Version of API Instance for pairing with Mule Application (e.g. v1)' )
        string( name: 'ORGANIZATION', defaultValue: 'sec-ops-2', description: 'Org or BusinessGroup to deploy the API' )
    }    
       
    stages {


        stage('echo') {
            steps {
                echo "$ANYPOINT_PLATFORM_CREDENTIALS_USR"
                echo "$ANYPOINT_PLATFORM_CREDENTIALS_PSW"
                echo "${params.ORGANIZATION}"
                echo "${env.ARTIFACT_ID}"
                echo "${params.API_VERSION}"
                echo "${env.MULE_RUNTIME_VERSION}"
                echo "${currentBuild.currentResult}"
            }
        }

        stage('Promote API to Testing') {
            when {
                expression { params.ENVIRONMENT == 'qa' }
            }
            steps {
                script {
                    echo "Promoting API from Development..."
                    sh """ newman run Promote-API.postman_collection-v7.json \
                                --env-var anypoint_username=$ANYPOINT_PLATFORM_CREDENTIALS_USR \
                                --env-var anypoint_password=$ANYPOINT_PLATFORM_CREDENTIALS_PSW \
                                --env-var anypoint_organisation=${params.ORGANIZATION} \
                                --env-var source_environment=dev \
                                --env-var target_environment=qa \
                                --env-var asset_id=${env.ARTIFACT_ID} \
                                --env-var product_version=${params.API_VERSION} \
                                --env-var anypoint_runtime=${env.MULE_RUNTIME_VERSION} \
                                --disable-unicode \
                                --color on \
                                -r htmlextra \
                                --reporters cli,json \
                                --reporter-json-export promote-api-output.json """
                    echo "Promoted API from Testing: ${currentBuild.currentResult}"
                }
            }    
            post {
                success {
                    echo "...Promote API from Development Succeeded for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                } 
                failure {
                    echo "...Promote API from Development Failed for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                }
            }
        } 


        stage('autodiscovery') {
            steps {
                script {
                    def postman_envs = readJSON file: 'promote-api-output.json'
                    def auto_discovery_id = postman_envs.environment.values.findIndexOf{ it.key == "auto_api_id" }
                    def contract_client_id = postman_envs.environment.values.findIndexOf{ it.key == "client_id" }
                    def contract_client_secret = postman_envs.environment.values.findIndexOf{ it.key == "client_secret" }
                    print "Autodiscovery API ID: " + postman_envs.environment.values[auto_discovery_id].value
                    print "Contract Client ID: " + postman_envs.environment.values[contract_client_id].value
                    print "Contract Client Secret: " + postman_envs.environment.values[contract_client_secret].value
                }
            }
        }
    }
}
