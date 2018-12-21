pipeline {
    agent any
	
    stages {
	    stage('Checkout Code"') {
			steps {
				checkout poll: false, scm: [$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/jigneshmehta1984/spring-boot-docker.git']]]
			}
		}
		stage('Build') {
			steps {
				sh '''
					file="./version"
			
					if [ -f "$file" ]
					then
					  echo "$file found."
					  while read content
					  do
						echo "${content}"
						if [ ! -z "$content" ]; then
						   version=$content
						   echo "${version}"
						fi
					  done < "$file"
					
					  echo "Version   = " ${version}
					else
					  echo "$file not found."
					fi
					
					mvn -Dbuild_version="${version}" -DforkCount=0 install dockerfile:build
				'''
			}
		}
		stage('Push Image') {
			steps {
				script {
				  // trim removes leading and trailing whitespace from the string
				  build_version = readFile('version').trim()
				  docker.withRegistry('https://registry.hub.docker.com', 'docker-registry') {
						docker.image("jigneshmehta1984/spring-boot-docker:${build_version}").push()
				  }
				}
			}
		}
        stage('Deploy') {
            steps {
				sh '''
			    file="./version"
        
                if [ -f "$file" ]
                then
                  echo "$file found."
                  while read content
                  do
                    echo "${content}"
                    if [ ! -z "$content" ]; then
                       BUILD_VERSION=$content
                       echo "${BUILD_VERSION}"
                    fi
                  done < "$file"
                
                  echo "Version   = " ${BUILD_VERSION}
                else
                  echo "$file not found."
                fi	
				
				REGION=ap-south-1
				REPOSITORY_NAME=docker-repository
				CLUSTER=spring-boot-docker-cluster
				
				echo "${REGION}"
				echo "${REPOSITORY_NAME}"
				echo "${CLUSTER}"
				
			    FAMILY=`sed -n 's/.*"family": "\\(.*\\)",/\\1/p' taskdef.json`
			    echo "${FAMILY}"
			    
			    NAME=`sed -n 's/.*"name": "\\(.*\\)",/\\1/p' taskdef.json`
			    echo "${NAME}"
			    
	        	SERVICE_NAME=${NAME}-service
	        	echo "${SERVICE_NAME}"
	        	
				REPOSITORY_URI=jigneshmehta1984/spring-boot-docker
				echo "${REPOSITORY_URI}"
				
				sed -e "s;%BUILD_VERSION%;${BUILD_VERSION};g" -e "s;%REPOSITORY_URI%;${REPOSITORY_URI};g" taskdef.json > ${NAME}-${BUILD_VERSION}.json
				
				cat ${NAME}-${BUILD_VERSION}.json
				
				aws ecs register-task-definition --family ${FAMILY} --cli-input-json file://${WORKSPACE}/${NAME}-${BUILD_VERSION}.json --region ${REGION}

				SERVICES=`aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .failures[]`
				echo "${SERVICES}"
				
				REVISION=`aws ecs describe-task-definition --task-definition ${NAME} --region ${REGION} | jq .taskDefinition.revision`
				echo "${REVISION}"
				
				DESIRED_COUNT=`aws ecs describe-services --services ${SERVICE_NAME} --cluster ${CLUSTER} --region ${REGION} | jq .services[].desiredCount`
				if [ ${DESIRED_COUNT} = "0" ]; then
				    DESIRED_COUNT="1"
				fi
				echo "DESIRED_COUNT ${DESIRED_COUNT}"
				
				aws ecs update-service --cluster ${CLUSTER} --region ${REGION} --service ${SERVICE_NAME} --task-definition ${FAMILY}:${REVISION} --desired-count ${DESIRED_COUNT}
				'''
				
			}
		}
	}	
}
