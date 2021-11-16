// Pipeline Script for BTC-ON-ECS Demo //
// Author: Prasanjit Singh //

pipeline {
  environment {
    // shouldn't need the registry variable unless you're not using dockerhub (Ex. Populate if using ECR)
    // registry = 'registry.hub.docker.com'
    //
    // change this HUB_CREDENTIAL to the ID of whatever jenkins credential has your registry user/pass
    // first let's set the docker hub credential and extract user/pass
    // we'll use the USR part for figuring out where are repository is
    HUB_CREDENTIAL = "DockerHub"
    // use credentials to set DOCKER_HUB_USR and DOCKER_HUB_PSW
    DOCKER_HUB = credentials("${HUB_CREDENTIAL}")
    // change repository to your DockerID
    REPOSITORY = "${DOCKER_HUB_USR}/bitcoin"
    VERSION = "0.21.0"
    AWS_ECR_REGION = 'us-east-1'
    AWS_ECS_SERVICE = 'btc-service'
    AWS_ECS_CLUSTER = 'btc-on-ecs'
    AWS_ECS_TASK_DEFINITION = 'btc-task'
    AWS_ECS_TASK_REVISION = '1'
  } // end environment

  agent any
  stages {

    stage('Checkout SCM') {
      steps {
        checkout scm
      } // end steps
    } // end stage "checkout scm"

    stage('Build image and tag with build number') {
      steps {
        script {
          dockerImage = docker.build REPOSITORY + ":${VERSION}"
        } // end script
      } // end steps
    } // end stage "build image and tag w build number"

    stage('Analyze with grype') {
      steps {
        // run grype with json output, use jq just to get severities,
        // concatenate all onto one line, if we find High or Critical
        // vulnerabilities, fail and kill the pipeline
        //
        // set -o pipefail enables the entire command to return the failure
        // in grype and still get the count of vulnerability types
        //
        // you can change this from "high" to "critical" if you want to see
        // the command succeed since pvnovarese/ubuntu_sudo_test doesn't (as
        // of today) have any critical vulns in it, just high.
        //
        script {
          try {
            // -f high --> fail if "high" or "critical" vulns detected
            // we don't really need to pipe into jq if we're just testing
            // for vulns, but this way we can get some output to provide
            // to devs as feedback.
            //
            sh 'set -o pipefail ; /usr/local/bin/grype -f high -q ${REPOSITORY}:${VERSION}'
          } catch (err) {
            // if scan fails, clean up (delete the image) and fail the build
            sh """
              echo "Vulnerabilities detected in ${REPOSITORY}:${VERSION}, cleaning up and failing build."
              docker rmi ${REPOSITORY}:${VERSION}
              exit 1
            """
          } // end try/catch
        } // end script
      } // end steps
    } // end stage "analyze with grype"

    stage('Re-tag as stable and push the image to registry') {
      steps {
        sh """
          docker push ${REPOSITORY}:${VERSION}
          docker tag ${REPOSITORY}:${VERSION} ${REPOSITORY}:stable
          docker push ${REPOSITORY}:stable
        """
      } // end steps
    } // end stage "retag as stable"



    stage('Deploy to ECS') {
      steps {
        withCredentials([string(credentialsId: 'AWS_EXECUTION_ROL_SECRET', variable: 'AWS_ECS_EXECUTION_ROL'),string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
          script {

    //Use the below when not handling task definitions via Terraform/IaC .......
            // updateContainerDefinitionJsonWithImageVersion()
            // sh("/usr/local/bin/aws ecs register-task-definition --region ${AWS_ECR_REGION} --family ${AWS_ECS_TASK_DEFINITION} --execution-role-arn ${AWS_ECS_EXECUTION_ROL} --requires-compatibilities ${AWS_ECS_COMPATIBILITY} --network-mode ${AWS_ECS_NETWORK_MODE} --cpu ${AWS_ECS_CPU} --memory ${AWS_ECS_MEMORY} --container-definitions file://${AWS_ECS_TASK_DEFINITION_PATH}")
            // def taskRevision = sh(script: "/usr/local/bin/aws ecs describe-task-definition --task-definition ${AWS_ECS_TASK_DEFINITION} | egrep \"revision\" | tr \"/\" \" \" | awk '{print \$2}' | sed 's/\"\$//'", returnStdout: true)
     //.............
            sh("/usr/local/bin/aws ecs update-service --cluster ${AWS_ECS_CLUSTER} --service ${AWS_ECS_SERVICE} --task-definition ${AWS_ECS_TASK_DEFINITION}:${AWS_ECS_TASK_REVISION}")
            //updates the ecs service
          }
        }
      }
    }


    stage('Clean up') {
      // delete the images locally from Jenkins Server
      steps {
        sh 'docker rmi ${REPOSITORY}:${VERSION} ${REPOSITORY}:stable'
      } // end steps
    } // end stage "clean up"

  } // end stages
} // end pipeline
