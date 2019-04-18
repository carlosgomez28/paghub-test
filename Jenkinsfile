/*
	Scripted Jenkinsfile pipeline code for PAGHUB
	Please include this file in your micro-service implementation and configure a Web Hook
	for that micro-service repository
	Version: 1.0
*/

/*
	- The pod template represents our jenkins agent or slave that will be launched as a kubernetes pod.
	  It needs to always include the JNLP (Java Network Launch Protocol) and the other container templates
      in order to run the pipeline commands.
	- The resource allocation was done based on performance test and best practices. Each agent pod will always occupy
      200 miliCPU in total (the sum of all cpu requests of each container), but sometimes each container could use
      more than the requested, therefore, a limit is specify. A container cannot pass the specified 
      resource (memory or cpu) limit. In addition, all container images are obtained from official repositories.
	- The hostPathVolume mounts the docker daemon from the master jenkins to the slaves.
	- the persistentVolumeClaim read and writes the repository dependencies in kubernetes storage (for performance
      improvement, without it, maven would have to download all repository dependencies for each build)
*/
podTemplate(label: 'super-pod',
	containers: [
		containerTemplate(
        	name: 'jnlp',
            image: 'jenkinsci/jnlp-slave:3.10-1-alpine',
			resourceRequestCpu: '50m',
			resourceLimitCpu: '550m',
			resourceRequestMemory: '64Mi',
    		resourceLimitMemory: '128Mi'
      	),
      	containerTemplate(
        	name: 'maven',
            image: 'maven:3.5.2-alpine',
            command: 'cat',
            ttyEnabled: true,
			resourceRequestCpu: '100m',
			resourceLimitCpu: '550m',
			resourceRequestMemory: '256Mi',
    		resourceLimitMemory: '500Mi'
      	),
		containerTemplate(
			name: 'docker',
         	image: 'gcr.io/cloud-builders/docker',
        	command: 'cat',
			ttyEnabled: true,
			resourceRequestCpu: '10m',
			resourceLimitCpu: '100m',
			resourceRequestMemory: '64Mi',
			resourceLimitMemory: '128Mi'
		),
		containerTemplate(
        	name: 'kubectl',
            image: 'gcr.io/cloud-builders/kubectl',
            command: 'cat',
           	ttyEnabled: true,
			resourceRequestCpu: '20m',
			resourceLimitCpu: '100m',
			resourceRequestMemory: '64Mi',
    		resourceLimitMemory: '128Mi'
		),
		containerTemplate(
			name: 'gcloud',
			image: 'gcr.io/cloud-builders/gcloud',
			command: 'cat',
            ttyEnabled: true,
			resourceRequestCpu: '20m',
			resourceLimitCpu: '100m',
			resourceRequestMemory: '64Mi',
    		resourceLimitMemory: '128Mi'
        ),
	],
	volumes: [
		hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
		persistentVolumeClaim(claimName: 'maven-repo', mountPath: '/home/jenkins/.m2'),
	]
)
{
	//For each agent execute the following pipeline
    node ('super-pod') {
		try{
			stage ('Checkout') {
				//check outs the configured GitHub repository
				checkout scm
			}
			stage ('Maven - unit test') {
				//The unit tests will only be performed for the dev branch
				if (env.BRANCH_NAME.toLowerCase() == 'dev'){
					//using maven container, execute...
					container('maven') {
						//the flag is important for maven to identify the repository folder in the agent
						//(saved in the persistent volume claim)
						sh("mvn -Dmaven.repo.local=/home/jenkins/.m2/ test")
		    		}
				}else {
					echo "Current branch: ${env.BRANCH_NAME.toLowerCase()} - Skipped stage."
				}
		    
			}
			stage ('Maven - package project') {
				if (env.BRANCH_NAME.toLowerCase() == 'dev'){
					container('maven') {
						sh("mvn -Dmaven.test.skip=true -Dmaven.repo.local=/home/jenkins/.m2/ clean package")
		    		}
				}else {
					echo "Current branch: ${env.BRANCH_NAME.toLowerCase()} - Skipped stage."
				}			    
			}
			stage ('Docker - build and push image') {
				//configure this secret file with the service account json file in Jenkins Credentials
				withCredentials([file(credentialsId: 'key-sa', variable: 'GC_KEY')]) {
					if (env.BRANCH_NAME.toLowerCase() == 'dev'){				
						container('docker'){
							//readMavenPom is available thanks to the pipeline utility steps plugin
							def pom = readMavenPom file: 'pom.xml'
							def image = pom.artifactId
							def version = pom.version
							def project = "pag-hub-ci-cd" 
							def tag = "${image}:${version}"
							//docker build and push authorization using the service account json key 
							//with the proper roles
							sh 'docker login -u _json_key -p "$(cat ${GC_KEY})" https://gcr.io'
							def customImage = docker.build("gcr.io/${project}/${tag}")
							customImage.push()
						}	
					}else {
						echo "Current branch: ${env.BRANCH_NAME.toLowerCase()} - Skipped stage."
					}						    			
				}
			}
			stage ('K8s - Deploy') {
				if (env.BRANCH_NAME.toLowerCase() != 'master'){
					withCredentials([file(credentialsId: 'key-sa', variable: 'GC_KEY')]) {
						container('gcloud'){
							//using the service account configured roles
							sh ("gcloud auth activate-service-account --key-file=${GC_KEY} --project pag-hub-dev")
							//connecting to the cluster
							sh ("gcloud container clusters get-credentials paghub-dev --zone northamerica-northeast1-a --project pag-hub-dev")					
						}
						container('kubectl'){
							sh("kubectl  apply -f deployment.yaml --namespace=${env.BRANCH_NAME.toLowerCase()}")
						}
					}		
				}else {
					echo 'Deploying in production...'
					//PRODUCTION CODE HERE...
				}		
			}    
    	}catch(e){
	    	//for any error in the pipeline, a notification will be sent
	    	mail body: "There was an error building the pipeline, see below for more details: <br><br> <b>Project:</b> ${env.JOB_NAME} <br><b>Branch:</b> ${env.BRANCH_NAME}<br><b>Build Number:</b> ${env.BUILD_NUMBER} <br> <b>URL of build:</b> ${env.BUILD_URL}<br> <b>Error:</b> ${e}", 
		    charset: 'UTF-8', 
		    from: '',
		    mimeType: 'text/html', 
		    subject: "FAILED: PAGHUB: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
		    to: "errors@performance.ca";  
			throw e
    	}
    }
}