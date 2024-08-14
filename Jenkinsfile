def getFolderName() {
  def array = pwd().split("/")
  return array[array.length - 2];
}
def agentLabel = "${env.JENKINS_AGENT == null ? "":env.JENKINS_AGENT}"

def parseJsonString(jsonString, key){
  def datas = readJSON text: jsonString
  String Values = writeJSON returnText: true, json: datas[key]
  return Values
}

def parseJsonArray(jsonString){
  def datas = readJSON text: jsonString
  return datas
}

def pushToCollector(){
  print("Inside pushToCollector...........")
    def job_name = "$env.JOB_NAME"
    def job_base_name = "$env.JOB_BASE_NAME"
    String generalProperties = parseJsonString(env.JENKINS_METADATA,'general')
    generalPresent = parseJsonArray(generalProperties)
    if(generalPresent.tenant != '' &&
    generalPresent.lazsaDomainUri != ''){
      echo "Job folder - $job_name"
      echo "Pipeline Name - $job_base_name"
      echo "Build Number - $currentBuild.number"
      sh """curl -k -X POST '${generalPresent.lazsaDomainUri}/collector/orchestrator/devops/details' -H 'X-TenantID: ${generalPresent.tenant}' -H 'Content-Type: application/json' -d '{\"jobName\" : \"${job_base_name}\", \"projectPath\" : \"${job_name}\", \"agentId\" : \"${generalPresent.agentId}\", \"devopsConfigId\" : \"${generalPresent.devopsSettingId}\", \"agentApiKey\" : \"${generalPresent.agentApiKey}\", \"buildNumber\" : \"${currentBuild.number}\" }' """
    }
}


pipeline {
	agent { label agentLabel }
  	environment {
        BRANCHES = "${env.GIT_BRANCH}"
        COMMIT = "${env.GIT_COMMIT}"
        RELEASE_NAME = "mysql"
        SERVICE_PORT = "${APP_PORT}"
        DOCKERHOST = "${DOCKERHOST_IP}"
        REGISTRY_URL = "${DOCKER_REPO_URL}"
		foldername = getFolderName()
		DEPLOYMENT_TYPE = "${DEPLOYMENT_TYPE == ""? "EC2":DEPLOYMENT_TYPE}"
		KUBE_SECRET = "${KUBE_SECRET}"
    }
    stages {
		stage('Initialization') {
			agent { label agentLabel }
			steps {
				script {
					TEMP_STAGE_NAME = "$STAGE_NAME"
					def job_name = "$env.JOB_NAME"
					print(job_name)
					def values = job_name.split('/')
					namespace_prefix = values[0].replaceAll("[^a-zA-Z0-9\\-\\_]+","").toLowerCase().take(50)
					namespace = "$namespace_prefix-$env.foldername".toLowerCase()
					service = values[2].replaceAll("[^a-zA-Z0-9\\-\\_]+","").toLowerCase().take(50)
					print("kube namespace: $namespace")
					print("service name: $service")
					env.namespace_name=namespace
					env.service=service
				}
			}
		}
        stage('Deploy') {
			agent { label agentLabel }
            steps {
				script {
					TEMP_STAGE_NAME = "$STAGE_NAME"
					if (env.DEPLOYMENT_TYPE == 'EC2') {

						sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker pull mysql:8.0.1"'
						sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker stop ${JOB_BASE_NAME}-mysql || true && docker rm ${JOB_BASE_NAME}-mysql || true"'
						sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker run --restart always --name ${JOB_BASE_NAME}-mysql -e MYSQL_ROOT_PASSWORD=password -p 3306:3306 -d mysql:8.0.1"'
						sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker pull phpmyadmin/phpmyadmin:latest"'
						sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker stop ${JOB_BASE_NAME}-pma || true && docker rm ${JOB_BASE_NAME}-pma || true"'
						sh 'ssh -o "StrictHostKeyChecking=no" ciuser@$DOCKERHOST "docker run --restart always --name ${JOB_BASE_NAME}-pma -d --link ${JOB_BASE_NAME}-mysql:db -p $SERVICE_PORT:80 phpmyadmin/phpmyadmin"'
					}
					if (env.DEPLOYMENT_TYPE == 'KUBERNETES') {

						withCredentials([file(credentialsId: "$KUBE_SECRET", variable: 'KUBECONFIG')]) {
								sh '''
									rm -rf kube
									mkdir -p kube
									cp "$KUBECONFIG" kube
									sed -i s+#SERVICE_NAME#+"$service"+g ./helm_chart/values.yaml ./helm_chart/Chart.yaml
									kubectl create ns "dev-mysql" || true
									helm upgrade --install mysql -n "dev-mysql" helm_chart --atomic --timeout 300s --set image.tag=8.0-debian-12  --set auth.rootPassword="password" --set primary.persistence.size="6Gi" --set primary.service.type="LoadBalancer"
									sleep 10
								'''
								script {
									env.temp_service_name = "$RELEASE_NAME-$service".take(63)
									def url = sh (returnStdout: true, script: '''kubectl get svc -n "$namespace_name" | grep "$temp_service_name" | awk '{print $4}' ''').trim()
									if (url != "<pending>") {
										print("##\$@\$ http://$url ##\$@\$")
									} else {
										currentBuild.result = 'ABORTED'
										error('Aborting the job as access url has not generated')
									}

								}
						}
					}
				}
            }
        }
    }

}
