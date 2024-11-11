pipeline {
    agent {
        label 'k8s-slave'
    }
    parameters {
        choice(name: 'buildOnly',
          choices: 'no\nyes',
          description: 'this will only build the application'
        )
        choice(name: 'scanOnly',
          choices: 'no\nyes',
          description: 'this will scan the application'
        )
        choice(name: 'dockerPush',
          choices: 'no\nyes',
          description: 'this will push the docker image'
        )
        choice(name: 'deployToDev',
          choices: 'no\nyes',
          description: 'this will deploy to the dev env'
        )
        choice(name: 'deployToStage',
          choices: 'no\nyes',
          description: 'this will deploy to the stage env'
        )
        choice(name: 'deployToProd',
          choices: 'no\nyes',
          description: 'this will deploy to the Prod env'
        )

    }
    environment{
        SERVICE_NAME = "user"
        POM_VERSION = readMavenPom().getVersion()
        POM_PACKAGING = readMavenPom().getPackaging()
        DOCKER_HUB = "docker.io/subhanicool01"
        DOCKER_CREDS = credentials('subhanicool01_docker_creds')
        SONAR_URL = "http://34.68.126.198:9000"
        SONAR_TOKEN = credentials('sonar_creds')
        PUBLIC_IP = "34.41.246.17"
    }
    tools {
        maven 'Maven-3.8.8'
        jdk 'JDK-17'
    }
    stages {
        stage("Build") {
          when {
            anyOf {
                expression {
                    params.buildOnly == 'yes'
                    params.dockerPush == 'yes'
                }
            }
          }
            steps {
               script {
                buildApp().call()
               }            
            }
        }
        stage('Unit tests') {
            when {
              anyOf {
                expression {
                    params.buildOnly == 'yes'
                    params.dockerPush == 'yes'
                }
              }
            }  
            steps {
                echo "performing the unit test cases"
                sh "mvn test"
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }
        // For these application not implemented the sonar
        stage('Docker format') {
            steps {
                echo "actual format: ${env.SERVICE_NAME}-${env.POM_VERSION}-${env.POM_PACKAGING}"
                //Need to have below formating
                //service-name-buildnumber-branchname.packing
                //pricecur-service-5-main.jar
                echo "custom format: ${env.SERVICE_NAME}-${currentBuild.number}-${BRANCH_NAME}-${env.POM_PACKAGING}"
            }
        }
        stage('Docker build') {
            when {
                expression {
                    params.dockerPush == 'yes'
                }
            }
            steps {
               script {
                 dockerBuildandPush().call()
               }
            }
        }
        stage('Deploy to dev') {
            when {
                expression {
                    params.deployToDev == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    Dockerdeploy('dev', '3736').call()
                }
            }
        }
        stage('Deploy to stage') {
             when {
                expression {
                    params.deployToStage == 'yes'
                }
            }
            steps {
                script {
                    imageValidation().call()
                    Dockerdeploy('stage', '4736').call()
                }
            }
        }
        stage('Deploy to Prod') {
            when {
                allOf {
                    anyOf {
                        expression {
                            params.deployToProd == 'yes'
                        }
                    }
                    anyOf {
                        branch 'release/*'
                    }
                }
            }
            steps {
                timeout(time: 100, unit: 'SECONDS') {
                    input message: "Deploying ${env.SERVICE_NAME} to Prod ???", ok: 'yes', submitter: 'Krishna'
                }
                script {
                    imageValidation().call()
                    Dockerdeploy('Prod', '5736').call()
                }
            }
        }
    }
}


def buildApp() {
    return {
        echo "Building the code for ${env.SERVICE_NAME}"
        echo "Now Starting the mvn packages"
        sh "mvn clean package -DskipTests=True"
        echo "Build is done" 
    }
}

def dockerBuildandPush() {
    return {
        sh """
            ls -la
            cp -f ${workspace}/target/${env.SERVICE_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} ./.cicd
            ls -la ./.cicd
            echo "********************** Build docker image ************************"
            docker build --force-rm --no-cache --pull --rm=true --build-arg JAR_SOURCE=${env.SERVICE_NAME}-${env.POM_VERSION}.${env.POM_PACKAGING} -t ${env.DOCKER_HUB}/${env.SERVICE_NAME}:${GIT_COMMIT} ./.cicd
            docker images
            echo "*********************** Docker push *****************************"
            docker login  -u ${env.DOCKER_CREDS_USR} -p ${env.DOCKER_CREDS_PSW}
            docker push  ${env.DOCKER_HUB}/${env.SERVICE_NAME}:${GIT_COMMIT}
        """    
    }
}

//this method is for if devloper was directly deploy the any env, there is no images are avaible.
def imageValidation() {
    return {
       println ("Pulling the docker image")
       try {
         sh "docker pull ${env.DOCKER_HUB}/${env.SERVICE_NAME}:${GIT_COMMIT}"
       } 
       catch (error) {
          println ("Oops docker image is not available")
          buildApp().call()
          dockerBuildandPush().call()
        }
    }
}

// this method is devloped for deploying a multiple environmnets.
def Dockerdeploy(env_Name, host_Port) {
    return {
    echo "***** deploying docker in $env_Name env *****"
            withCredentials([usernamePassword(credentialsId: 'docker_dev_vm_creds', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                // some block
                // with the help of these block, this slave will be connecting to docker vm machine to executing containers.
                //sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@34.41.49.154 hostname -i"
                // docker login and pull

            script {
                // pulling the container
                echo "pulling the container"

                sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${env.PUBLIC_IP} docker pull ${env.DOCKER_HUB}/${env.SERVICE_NAME}:${GIT_COMMIT}"
                try {
                    // stop the container 
                    echo "stopping the container"
                    sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${env.PUBLIC_IP} docker stop ${env.SERVICE_NAME}-$env_Name"

                    // remove the container
                    echo "removing the container"
                    sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${env.PUBLIC_IP} docker rm ${env.SERVICE_NAME}-$env_Name"

                } catch(err) {
                    echo "Caught the error: $err"
                }
                // docker create a conatainer
                   echo "creating a new-container"
                   sh "sshpass -p ${PASSWORD} -v ssh -o StrictHostKeyChecking=no ${USERNAME}@${env.PUBLIC_IP} docker run -d -p $host_Port:5736 --name ${env.SERVICE_NAME}-$env_Name ${env.DOCKER_HUB}/${env.SERVICE_NAME}:${GIT_COMMIT}"


            } 
            }
    }

}

//  /home/subhanicool01/jenkins/workspace/pricecut-service_main/target/price-cut-service-0.0.1-SNAPSHOT.jar
//   git remote set-url origin https://{subhanicool01}:{Subhani@5736}@github.com/{subhanicool01}/project.git
//   after install the docker we need to fix one issue for permission denied. To add user for docker group so we write a module on ansible playbook.

//docker build --force-rm --no-cache --pull --rm=true 
// --force-rm - forcefully remove
// --no-cache - no cacheing 
// --pullm --rm=true - to pull the remote directory

// soanr-Docker_cicd_envs
  //  1. sonar scan is ready for selected service.
  //  2. soanr is working, but dev pushes the code in multiple times so we need to findout the how many bucks are there.
  //  3. so we are implement quality-gates for project to verify how many code smells are there. if cross fail, if not crosses then pass.
  //  4. if build is done, but the code is some codesmiles are not proper way.
  //  5. So we are using sonar quality gates in jenkins.
  //  6. using some methods to that sonar stage is populating some colours.
  //  7. so used one pluging sonarqube scanner && install
  //  8. after that add sonarqube details in manage-jenkins-system-
// Docker
  // now code is avaible how to deploy in real time used k8s through nginx 
  // now we are using another ways to deploy.
  // 1. deploy to docker container  2. using shared libraries 3. k8s- deployment   4. deploy thorough helm charts
  // 2. now we are following docker way right. In these way using dev-env deploy, stage-env deply, prod-env-deploy.
  // 3. So these way docker is aviable in slave do you want to execute all env in slave machine? "NO"
  // 4. Each Env is saparate machine and having docker. In real time to deploy k8s way.

  // eureka-server runs at port no 8761
  // I will configure env's in a way they will have diff host ports
  // dev ---> 5761 (HP)
  // stage ---> 6761 (HP)
  // test  ---> 7761 (HP)
  // Prod   ---> 8761 (HP) make sure these all Host ports are opend firewallrules

 // Method calling
  // if we are uesd one env is directly used, but using multiple env's, added all the code into all environments.
  // so that's way actual code will call the multiple env.

 // parameters
   // when the dev team apply build now all the stages are executable prod also.
   // so these is big mistake, so chnage the code into parametaraized.

// Stage permission
   // if some feature branches are merge into master and deployed into dev and stage env's.
   // if dev team will build the code, these code will deployed into Production env also.
   // These is major mistake for devops role, no feature,bugs,hotfix are deployed into prod.
   // And dev team never deploy the Prod env directly. Devops team handled and Permissioned.
   // to satisfy the when condition.

 // Approvals
   // if code is ready to deploy into prod, who deploy the prod? dev "No"
   // dev team only deploy dev env's only.
   // instad of approval of Manage or DevOps person.
   // need add a users to call to approve the Prod.
   // Go-to manage-jenkins - users- add a users.

 // Microservices aded
  // 1. Now we are working with eureka-server. These eureka-server will act as a discovery-server(registry)
  // 2. these registry will pick up or cummunicate the all microservices.
  // 3. Now we add a another two services and one node application and one database
  // 4. 1. service-1 2. service-2 3.Node-application 4. database
  //
