pipeline {

	agent any 

	environment {
	   APP_NAME="d-api"
	   IMAGE_NAME="ohadd306/d-api"
	   IMAGE_TAG= "${BUILD_NUMBER}"
           PRD_PORT="8000"
           ROLLBACK_FILE="/var/lib/jenkins/rollback/d_api_python/last_successful_tag" 

	}

	stages {
	
		stage('checkout') {
		   steps {
                     checkout scm 
                   }
		}
		stage('unit tests') {
                   steps { 
                     echo "running unit tests"
                     sh '''
                        python3 -m pytest -v
                     ''' 
                   }
		}
		stage('build docker image') {
                    steps {
                      sh '''
                      docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                      '''
		    }
		}
                stage('test docker image') {
		   steps {
                       echo "test docker image"
			sh '''
			docker run -d --name ${APP_NAME}-test -p 8001:8000 ${IMAGE_NAME}:${IMAGE_TAG}
                        '''
                        echo "wait for app to start"
                        sleep 3 
                        echo "starting docker image healthcheck"
			sh '''
                        curl -f http://localhost:8001/health
                        curl -f  -X POST  -H "Content-Type: application/json" -d '{"product_id":1,"quantity":3}' http://localhost:8001/orders/
                        curl -f http://localhost:8001/products/1
			''' 
			
                   }

                   post {
			always {
                           echo "remove docker test container"
                           sh '''
                           docker stop ${APP_NAME}-test ||  true
                           docker rm ${APP_NAME}-test  ||  true
                           '''
 			}
			
	 	   }

                }

                stage('security scan with trivy')  {
                    steps {
                        echo "runnng securityu scan on image"
                        sh '''
 			   trivy image --severity CRITICAL --ignore-unfixed --exit-code 1 ${IMAGE_NAME}:${IMAGE_TAG}
                        '''
 
                    }
                }
        stage('Push to Docker Hub') {

            steps {

                echo "Logging into Docker Hub..."

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | \
                        docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin

                        docker push \
                            ${IMAGE_NAME}:${IMAGE_TAG}

                        docker logout
                    '''
                }
            }
        }
	stage('deploy on stg') {
	    steps{
		echo "start stg deployment"
		sh '''
                docker stop ${APP_NAME}-stg || true 
                docker rm ${APP_NAME}-stg  || true 
		docker pull ${IMAGE_NAME}:${IMAGE_TAG}
                docker run -d --name ${APP_NAME}-stg  -p 8002:8000  ${IMAGE_NAME}:${IMAGE_TAG}
                '''
  	    }
        }
	stage("stg health check tests") {
	    steps{
		echo "test health check on stg"
		sleep 5
		   sh '''
		   curl -f http://localhost:8002/health 
		   curl -f http://localhost:8002/products/1
		   
		   '''
		} 
	}
	stage("approve PRD deployment")  {
	   steps {
	      input(
		message: """
                Production Deployment Approval

                Application: ${IMAGE_NAME}
                Version: ${IMAGE_TAG}
                Build: ${BUILD_NUMBER}

                Deploy this version to PRODUCTION?
            """,
            ok: "Deploy ${IMAGE_TAG}",
            submitter: "ohad"	

	      )		
	   }
	}
	stage("deploy on PRD") {
	    steps {
		sh '''
		docker stop  ${APP_NAME} || true
		docker rm -f ${APP_NAME}  || true
		docker pull ${IMAGE_NAME}:${IMAGE_TAG}
		docker run -d --name ${APP_NAME} -p ${PRD_PORT}:8000 ${IMAGE_NAME}:${IMAGE_TAG}
		 
		'''
	    }
	}
	stage("test prd deployment health"){ 
	    steps {
	       echo "test prd deployment"
	       sh '''
	       	   for i in {1..3} ; do 
	              if curl -fsS http://localhost:8000/health ; then
			echo "health check test path"
		         if curl -fsS http://localhost:8000/products/1 ; then
			   echo "products test path successfuly" 
                             if curl -f -X POST -H "Content-Type: application/json" -d '{"product_id":1,"quantity":3}' http://localhost:8000/orders/ ; then
			       echo "process order path"
                               exit 0 
                             fi
			 fi
                      fi
                        echo "after $i attempts app is not ready yet"
			echo "wait for app to start"
			sleep 3 
		done
                echo "production deployment tests failed"
		exit 1
		    
	       '''	
	    }
	}
	stage("record successful deployment") {
		steps {
       		   writeFile(
                   file: env.ROLLBACK_FILE,
                   text: "${BUILD_NUMBER}"
                   )             
		}
	}
	
   }
       	


}
