import groovy.json.JsonSlurper
library 'jenkins_shared_libraries@1.0.11'

properties([
  parameters([
    string(name: 'IMSBUILD', description: 'IMSBUILD number'),
    string(name: 'IMS_BRANCH', description: 'IMS_BRANCH number', defaultValue: "7_6_0-LINA"),
    string(name: 'TAGNAME', description: 'IMSBUILD tag'),
    booleanParam(name: 'BQT', description: 'Enable/Disable BQT'),
    booleanParam(name: 'BAT', description: 'Enable/Disable BAT'),
    booleanParam(name: 'STS', description: 'Enable/Disable STS'),
    booleanParam(name: 'SRTS', description: 'Enable/Disable SRTS'),
    string(name: 'RECIPIENT', defaultValue: 'ankushk2@cisco.com', description: 'Recipient of any email notifications'),
    string(name: 'SENDER', defaultValue: 'ankushk2@cisco.com', description: 'Sender of any email notifications'),
    string(name: 'EMAIL', description: 'If there is a failure send an email to this address', defaultValue: "ankushk2@cisco.com"),
    string(name: 'ASAPROJECT', description: 'Specify the asa project for pointer update', defaultValue: "zambia"),
    string(name: 'REMOTE_JENKINS_URL', description: 'Specify the name of a remote jenkins server', defaultValue: "https://firepower-build.service.ntd.ciscolabs.com"),
    string(name: 'UPSTREAM_STATUS', description: 'The status of the upstream job that started this one.', defaultValue: "SUCCESS"),
    ]),
    buildDiscarder(logRotator(numToKeepStr:'50',daysToKeepStr: '7',artifactDaysToKeepStr: '7',artifactNumToKeepStr: '50')),
])

REMOTE_JOB_TOKEN = "NotARefrigerator"
REMOTE_AUTH = "firepower-build-token"

def imsBuild = "${params.IMSBUILD}"

ignoreList = ["${JOB_NAME.tokenize("/").last()}", "cim_trigger"]
parallelJobs = [:]
buildResults = [:]

timestamps {
    node ('ful-jenkins-slave1.cisco.com') {

        try {
            stage ('Local Pointer Update') {
                sh """ 
                echo $IMSBUILD > ../sit_build.txt
                cat ../sit_build.txt
                """
            }

            if(params.UPSTREAM_STATUS == "SUCCESS"){
                if(BAT.toBoolean()){
                    //Get url from job name
                    def url = JOB_URL.substring(0,JOB_URL.lastIndexOf("/job/"))
                
                    //Get list of all qualified jobs in the folder
                    def jobs = getAllJobs(url)
                    println "$jobs"
                    // def jobss = ['test_job_one','test_job_two']
                
                    //For every job add them to the map of jobs to execute
                    for(job in jobs){
                        //Have to declare a variable because job.name doesn't expand correctly inside the map
                        def jobName = job.name
                        parallelJobs[jobName] = {
                            //Each job will have it's own stage
                            stage ("Build ${jobName}") {
                                try{
                                    //propagate flag must be set to true to get the status of the downstream job.
                                    buildResults[jobName] = build(job: jobName, parameters: [[$class: 'StringParameterValue', name: 'BUILD', value: imsBuild]], propagate: false, wait: true).result
                                } catch (e){
                                    //In case of error we must still capture the result. 
                                    buildResults[jobName] = 'FAILURE'
                                    error e
                                }
                            }
                        }
                    }
                
                    //execute all jobs in the map in parallel
                    parallel parallelJobs
                }
            }
                                
        } catch (e) {
            
            //throw the exception
            throw e
        } finally {
            stage ("Trigger Pointer Update Job") {
                def wrapperJobStatus = getJobStatus("IMS/${params.IMS_BRANCH}/WRAPPER")
                println(wrapperJobStatus)

                def failure = checkForFailure()
                echo "fail: ${failure}"
                echo "post build actions"

                if(params.UPSTREAM_STATUS == "SUCCESS" && wrapperJobStatus == "SUCCESS" && failure == false){
                    //setting the build result to success
                    buildResults = "SUCCESS"
                    //print build results on console
                    echo("buildResults is ${buildResults}")
					
                    ASAVERSION=getASAVersion()
                    
                    if ("${params.IMS_BRANCH}".contains("7_4_1-LINA")) {
                        SF_PROJECT = "IMS_7_4_1_MAIN"
                    } else if ("${params.IMS_BRANCH}".contains("7_6_0-LINA")) {
                        SF_PROJECT = "IMS_7_6_MAIN"
                    }

					if(!ASAVERSION.contains("Error")){
						try {
							//parameters passed to the remote job
							def payload = ["ASAPROJECT=${params.ASAPROJECT}", "ASAVERSION=${ASAVERSION}", "SF_PROJECT=${SF_PROJECT}"]
							println(payload)
							//set the remote job name
							def REMOTE_JOB_URL = "IMS/view/RelEng/job/Pointer_Update_P4/job/IMS_ASA_pointer_update/"
							echo "Triggering remote job: ${REMOTE_JOB_URL}..."
							//call the remote job with parameters
							remoteBuild.build(
								params.EMAIL, 
								params.REMOTE_JENKINS_URL,
								REMOTE_JOB_URL,
								REMOTE_JOB_TOKEN,
								REMOTE_AUTH,
								payload
							)
						} catch (e){
							error "Caught Exception ${e}"
						}
					}

                    notifyBuildStatus(buildResults)
                } else {
                    //setting the build result to failure
                    buildResults = "FAILURE"
                    //print build results on console
                    echo("buildResults is ${buildResults}")

                    //notifyBuildStatus(buildResults)
                }

                //cleanup directory post completion
                deleteDir()
            }
        }
    }
}

def getJobStatus(jobName){
	def job_url = jobName.replace("/","/job/")
    def json = sh( script: "curl ${REMOTE_JENKINS_URL}/job/${job_url}/lastBuild/api/json", returnStdout: true)
    def result = json.split("\"result\":\"")[1].split("\",")[0] 
    return result
}


//triggers an email depending on the status of the build
def notifyBuildStatus(buildStatus) {
    def details = ""
    def mailSubject = ""
    if(buildStatus == "SUCCESS") {
        //body of the mail for green build
        details = """<STYLE>BODY,P {  font-family:Verdana,Helvetica,sans serif;  font-size:11px;  color:black;}h1 { color:black; }h2 { color:black; }h3 { color:black; }</STYLE>
                     <BODY>
                     <B style="font-size: 250%;"> <IMG SRC="${REMOTE_JENKINS_URL}/static/e59dfe28/images/32x32/blue.gif"/>BUILD SUCCESSFUL</B>
                     <p style="font-size: 120%;">Successful: Job ${env.JOB_NAME} [${BUILD_NUMBER}]</p>
                     <p style="font-size: 120%;">Build: ${BUILD_NUMBER}</p>
                     <p style="font-size: 120%;">Host: ${NODE_NAME}</p>
                     <p style="font-size: 120%;">Build URL: ${env.BUILD_URL}</p>
                     <p style="font-size: 120%;">Check console output at: &quot;<a href='${env.BUILD_URL}/console'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&quot;</p>
                    </BODY>"""
        mailSubject = "${env.JOB_NAME} build job SUCCESSFUL."
    } else {
        //body of the mail for red buildStatus
        details = """<STYLE>BODY,P {  font-family:Verdana,Helvetica,sans serif;  font-size:11px;  color:black;}h1 { color:black; }h2 { color:black; }h3 { color:black; }</STYLE>
                      <BODY>
                      <B style="font-size: 250%;"> <IMG SRC="${REMOTE_JENKINS_URL}/static/e59dfe28/images/32x32/red.gif"/>BUILD FAILED</B>
                      <p style="font-size: 120%;">Failed: Job ${env.JOB_NAME} [${BUILD_NUMBER}]</p>
                      <p style="font-size: 120%;">Build: ${BUILD_NUMBER}</p>
                      <p style="font-size: 120%;">Host: ${NODE_NAME}</p>
                      <p style="font-size: 120%;">Build URL: ${env.BUILD_URL}</p>
                      <p style="font-size: 120%;">Check console output at: &quot;<a href='${env.BUILD_URL}/console'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&quot;</p>
                      </BODY>"""
        mailSubject = "${env.JOB_NAME} build job FAILED!"
    }
    try{
        emailext subject: mailSubject,
            body: details,
            mimeType: 'text/html',
            to: "${params.RECIPIENT},sowmnaga@cisco.com,alnatara@cisco.com,susboppa@cisco.com,lfenu@cisco.com",
            replyTo: params.RECIPIENT,
            from: params.SENDER
    }catch (e) {
		log.err("Unable to send email!")
		exceptionDetails(e)
	}
}

//Gets all Jobs (not subfolders) in a folder
def getAllJobs(url){
    def jobs = []
    //Get job list from API
    def json = sh(script: "curl -g ${url}/api/json?tree=jobs[name,url,color]", returnStdout: true).split("jobs\":")[1].replace("[","").replace("]}","")
    
    //Process json entries one by one
    for(str in json.split("},")){
        def entry = "" + str + "}"
        def job = readJSON text: entry.trim()
        //echo "Class: " + job._class + ", Name: " + job.name
        //Ignore folders
        if(!job._class.contains("Folder")){
            //ignore jobs in the ignore list
            def ignore = false
            for(i in ignoreList){
                if (i == job.name){
                    ignore = true
                    break
                }
            }
            if (!ignore && !job.color.contains("disabled")){
                jobs.add(job)
            }
        }
    }
    return jobs
}

//Get current ASA Version which need to be updated 
def getASAVersion(){
	try {
		def json = sh(script: "curl -g https://raum.cisco.com:8089/json/cp/${params.IMS_BRANCH}/0/now", returnStdout: true)
    
		def parser = new JsonSlurper()
		def new_json = parser.parseText(json)
    
		println("Current ASA version ->")
		println(new_json[2][1].split(" ")[0])
		return new_json[2][1].split(" ")[0]
	} catch (e){
		echo "Error Retrieving ASA Version!"
		return "Error"
	}
}

//Check if any of the build results failed. Returns true if a filure is found.
def checkForFailure(){
    def failure = false
    for (def key in buildResults.keySet()){
        echo "Job: ${key}: ${buildResults[key]}"
        if(buildResults[key] == 'FAILURE'){
            failure = true
            break
        }
    }
    return failure
}
