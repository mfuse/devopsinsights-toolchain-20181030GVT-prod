#!groovy
/*
    This is an sample Jenkins file for the Weather App, which is a node.js application that has unit test, code coverage
    and functional verification tests, deploy to staging and production environment and use IBM Cloud DevOps gate.
    We use this as an example to use our plugin in the Jenkinsfile
    Basically, you need to specify required 4 environment variables and then you will be able to use the 4 different methods
    for the build/test/deploy stage and the gate 
 */
pipeline {
    agent {
        label env.DEFAULT_JENKINS_AGENT
    }
    environment {
        // You need to specify 4 required environment variables first, they are going to be used for the following IBM Cloud DevOps steps
       // IBM_CLOUD_DEVOPS_CREDS = credentials('BM_CRED')
      //  IBM_CLOUD_DEVOPS_ENV = 'staging'
        IBM_CLOUD_DEVOPS_API_KEY = credentials('API_KEY_PROD')
        IBM_CLOUD_DEVOPS_ORG = 'fuse@jp.ibm.com'
        IBM_CLOUD_DEVOPS_APP_NAME = 'WheatherApp-20181011GVTアプリ①'
        IBM_CLOUD_DEVOPS_TOOLCHAIN_ID = '07d8353c-20a2-474a-9614-9c39a195952e'
        IBM_CLOUD_DEVOPS_WEBHOOK_URL = 'https://jenkins:30ebc25c-03a9-4baf-9987-fa4f270b72c9:6d9c8d94-2758-44a7-9739-9ba2cd33bb89@devops-api.ng.bluemix.net/v1/toolint/messaging/webhook/publish'
    }
    tools {
       nodejs 'recent'
   }
    stages {
        stage('ビルド') {
            environment {
                // get git commit from Jenkins
                GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                GIT_BRANCH = 'master'
                GIT_REPO = "https://github.com/mfuse/devopsinsights-toolchain-20181011GVT"
            }
            steps {
                checkout scm
                sh 'npm --version'
                sh 'npm install'
                sh 'grunt dev-setup --no-color'
            }
            // post build section to use "publishBuildRecord" method to publish build record
            post {
                success {
                    publishBuildRecord gitBranch: "${GIT_BRANCH}", gitCommit: "${GIT_COMMIT}", gitRepo: "${GIT_REPO}", result:"SUCCESS"
                }
                failure {
                    publishBuildRecord gitBranch: "${GIT_BRANCH}", gitCommit: "${GIT_COMMIT}", gitRepo: "${GIT_REPO}", result:"FAIL"
                }
            }
        }
        stage('単体テストとコードカバレッジ') {
            steps {
                sh 'grunt dev-test-cov --no-color -f'
            }
            // post build section to use "publishTestResult" method to publish test result
            post {
                always {
                 //   publishTestResult type:'ut2', fileLocation: './mochatest.json'
                    publishTestResult type:'unittest', fileLocation: './mochatest.json'
                    publishTestResult type:'code', fileLocation: './tests/coverage/reports/coverage-summary.json'
                }
            }
        }
               stage('SCM') {
            steps {
                git 'https://github.com/mfuse/devopsinsights-toolchain-20181030GVT-prod.git'
            }
        }
        stage ('SonarQube analysis') {
            steps {
                script {
                    // requires SonarQube Scanner 2.8+
                    def scannerHome = tool 'SonarQube Scanner GVT';
                   
                    withSonarQubeEnv('SonarQube GVT') {

                        env.SQ_HOSTNAME = SONAR_HOST_URL;
                        env.SQ_AUTHENTICATION_TOKEN = SONAR_AUTH_TOKEN;
                        env.SQ_PROJECT_KEY = "devopsinsights-toolchain-20181030GVT-prod";
                       

                      sh "${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=${SQ_PROJECT_KEY} \
                                -Dsonar.sources=.";
                         //       -Dsonar.organization=default-organization";
                    }
                }
            }
        }
        stage ("SonarQube Quality Gate") {
             steps {
                script {

                    def qualitygate = waitForQualityGate()
                    if (qualitygate.status != "OK") {
                        error "Pipeline aborted due to quality gate coverage failure: ${qualitygate.status}"
                    }
                }
             }
             post {
                always {
                    publishSQResults SQHostURL: "${SQ_HOSTNAME}", SQAuthToken: "${SQ_AUTHENTICATION_TOKEN}", SQProjectKey:"${SQ_PROJECT_KEY}"
                }
             }
        }
                stage('Security scan') {
            steps {
             sh 'echo '
            }
                 // post build section to use "publishTestResult" method to publish test result
            post {
                always {
                   publishTestResult type:'staticsecurityscan', fileLocation: './tests/asoc/gvt_2018-07-25_14-52-49.xml'
                 //   publishTestResult type:'staticsecurityscan', fileLocation: './tests/asoc/IDSInventory.xml'
                   publishTestResult type:'dynamicsecurityscan', fileLocation: './tests/asoc/gvt_2018-07-25_14-56-43.xml'
                }
            }
        }
        stage('ステージングにデプロイ') {
            steps {
                // Push the Weather App to Bluemix, staging space
                sh '''
                        echo "CF ログイン..."
                        cf api https://api.ng.bluemix.net
                     #  cf login -u $IBM_CLOUD_DEVOPS_CREDS_USR -p $IBM_CLOUD_DEVOPS_CREDS_PSW -o $IBM_CLOUD_DEVOPS_ORG -s ステージング
                        cf login -u apikey -p $IBM_CLOUD_DEVOPS_API_KEY -o $IBM_CLOUD_DEVOPS_ORG -s ステージング
                        
                        echo "デプロイ中...."
                     #  export CF_APP_NAME="staging-$IBM_CLOUD_DEVOPS_APP_NAME"
                        export CF_APP_NAME="staging-gvt20181030"                  
                        cf delete $CF_APP_NAME -f
                        cf push $CF_APP_NAME -n $CF_APP_NAME -m 64M -i 1

                        # use "cf icd --create-connection" to enable traceability
                       # cf icd --create-connection $IBM_CLOUD_DEVOPS_WEBHOOK_URL $CF_APP_NAME

                        export APP_URL=http://$(cf app $CF_APP_NAME | grep urls: | awk '{print $2}')
                    '''
            }
            // post build section to use "publishDeployRecord" method to publish deploy record and notify OTC of stage status
            post {
                success {
            //        publishDeployRecord environment: "STAGING", appUrl: "http://staging-${IBM_CLOUD_DEVOPS_APP_NAME}.stage1.mybluemix.net", result:"SUCCESS"
                    publishDeployRecord environment: "STAGING", appUrl: "http://staging-gvt20181030.mybluemix.net", result:"SUCCESS"
                    // use "notifyOTC" method to notify otc of stage status
//                  notifyOTC stageName: "ステージングにデプロイ", status: "SUCCESS"
//                  notifyOTC stageName: "ステージングにデプロイ", status: "成功 😊"

                }
                failure {
               //     publishDeployRecord environment: "STAGING", appUrl: "http://staging-${IBM_CLOUD_DEVOPS_APP_NAME}.stage1.mybluemix.net", result:"FAIL"
                    publishDeployRecord environment: "STAGING", appUrl: "http://staging-gvt20181030.mybluemix.net", result:"FAIL"
                    
                    // use "notifyOTC" method to notify otc of stage status
            //      notifyOTC stageName: "ステージングにデプロイ", status: "FAIL"
            //        notifyOTC stageName: "ステージングにデプロイ", status: "失敗 😢"
                }
            }
        }
        stage('FVT') {
            //set the APP_URL as the environment variable for the fvt 
            environment {
              //  APP_URL = "http://staging-${IBM_CLOUD_DEVOPS_APP_NAME}.stage1.mybluemix.net"
                APP_URL = "http://staging-gvt20181030.mybluemix.net"
            }
            steps {
                sh 'grunt fvt-test --no-color -f'
            }
            // post build section to use "publishTestResult" method to publish test result
            post {
                always {
                    publishTestResult type:'fvt', fileLocation: './mochafvt.json', environment: 'STAGING'
                }
            }
        }
        stage('ゲート') {
            steps {
                // use "evaluateGate" method to leverage IBM Cloud DevOps gate
           //     evaluateGate policy: 'Weather App Policy', forceDecision: 'true'
          evaluateGate policy: '天気予報アプリポリシー①', forceDecision: 'true'
            }
        }
        stage('実稼働にデプロイ') {
            steps {
                // Push the Weather App to Bluemix, production space
                sh '''
                        echo "CF ログイン..."
                        cf api https://api.ng.bluemix.net
                      # cf login -u $IBM_CLOUD_DEVOPS_CREDS_USR -p $IBM_CLOUD_DEVOPS_CREDS_PSW -o $IBM_CLOUD_DEVOPS_ORG -s 実稼働
                        cf login -u apikey -p $IBM_CLOUD_DEVOPS_API_KEY -o $IBM_CLOUD_DEVOPS_ORG -s 実稼働

                        echo "デプロイ中...."
                        export CF_APP_NAME="prod-$IBM_CLOUD_DEVOPS_APP_NAME"
                        cf delete $CF_APP_NAME -f
                        cf push $CF_APP_NAME -n $CF_APP_NAME -m 64M -i 1

                        # use "cf icd --create-connection" to enable traceability
                        #cf icd --create-connection $IBM_CLOUD_DEVOPS_WEBHOOK_URL $CF_APP_NAME

                        export APP_URL=http://$(cf app $CF_APP_NAME | grep urls: | awk '{print $2}')
                    '''
            }
            // post build section to use "publishDeployRecord" method to publish deploy record and notify OTC of stage status
            post {
                success {
                    publishDeployRecord environment: "PRODUCTION", appUrl: "http://prod-${IBM_CLOUD_DEVOPS_APP_NAME}.mybluemix.net", result:"SUCCESS"
                    // use "notifyOTC" method to notify otc of stage status
                  //notifyOTC stageName: "実稼働にデプロイ", status: "SUCCESS"
              //      notifyOTC stageName: "実稼働にデプロイ", status: "成功 😊"
                }
                failure {
                    publishDeployRecord environment: "PRODUCTION", appUrl: "http://prod-${IBM_CLOUD_DEVOPS_APP_NAME}.mybluemix.net", result:"FAIL"
                    // use "notifyOTC" method to notify otc of stage status
                //  notifyOTC stageName: "実稼働にデプロイ", status: "FAIL"
                //    notifyOTC stageName: "実稼働にデプロイ", status: "失敗 😪"
                }
            }
        }
    }
}
