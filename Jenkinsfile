
import groovy.json.JsonSlurper
import groovy.json.JsonOutput; 

def CONFIGDETAILS 
String MISSINGUSERS = ""
def INSTANCEARN = ""
def TRAGETINSTANCEARN = ""
String PRIMARYUSERS = ""
String TARGETUSERS = ""
String PRIMARYRPS = ""
String TARGETRPS = ""
String PRIMARYSECPROS = ""
String TARGETSECPROS = ""
String PRIMARYHRCHY = ""
String TARGETHRCHY = ""

pipeline {
    agent any
    stages {
        stage('git repo & clean') {
            steps {
                script{
                   try{
                      sh(script: "rm -r ac-queue-syncronization", returnStdout: true)    
                   }catch (Exception e) {
                       echo 'Exception occurred: ' + e.toString()
                   }                   
                   sh(script: "git clone https://github.com/ramprasadsv/ac-queue-syncronization.git", returnStdout: true)
                   sh(script: "ls -ltr", returnStatus: true)
                   CONFIGDETAILS = sh(script: 'cat parameters.json', returnStdout: true).trim()
                   def config = jsonParse(CONFIGDETAILS)
                   INSTANCEARN = config.primaryInstance
                   TRAGETINSTANCEARN = config.targetInstance
                }
            }
        }
        
        stage('List all Resources') {
            steps {
                echo "List all Resources in both instance "
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                    script {
                        
                        PRIMARYUSERS =  sh(script: "aws connect list-users --instance-id ${INSTANCEARN} ", returnStdout: true).trim()
                        echo PRIMARYUSERS
                        TARGETUSERS =  sh(script: "aws connect list-users --instance-id ${TRAGETINSTANCEARN} ", returnStdout: true).trim()
                        echo TARGETUSERS
                        
                        PRIMARYRPS =  sh(script: "aws connect list-routing-profiles --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYRPS
                        TARGETRPS =  sh(script: "aws connect list-routing-profiles --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETRPS 
                        
                        PRIMARYSECPROS =  sh(script: "aws connect list-security-profiles --instance-id ${INSTANCEARN} ", returnStdout: true).trim()
                        echo PRIMARYSECPROS
                        TARGETSECPROS =  sh(script: "aws connect list-security-profiles --instance-id ${TRAGETINSTANCEARN} ", returnStdout: true).trim()
                        echo TARGETSECPROS
                      
                        PRIMARYHRCHY = sh(script: "aws connect list-user-hierarchy-groups --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYHRCHY
                        TARGETHRCHY = sh(script: "aws connect list-user-hierarchy-groups --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETHRCHY                      
                    }
                }
            }
        }
        
        stage('Find missing queues') {
            steps {
                script {
                    echo "Find missing queues in the target instance"
                    def pl = jsonParse(PRIMARYQUEUES)
                    def tl = jsonParse(TARGETQUEUES)
                    int listSize = pl.QueueSummaryList.size() 
                    println "Primary list size $listSize"
                    for(int i = 0; i < listSize; i++){
                        def obj = pl.QueueSummaryList[i]
                        String qcName = obj.Name
                        String qcId = obj.Id
                        boolean qcFound = checkList(qcName, tl)
                        if(qcFound == false) {
                            println "Missing Name : $qcName Id : $qcId"                                                              
                            MISSINGQC = MISSINGQC.concat(qcId).concat(",")                                
                        }
                    }
                }
                echo "Missing list in the target instance -> ${MISSINGQC}"
            }
        }
        
        stage('Create the missing queues') {
            steps {
                echo "Create the missing queues in the target instance "                
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {   
                    script {
                        if(MISSINGQC.length() > 1 ){
                            def qcList = MISSINGQC.split(",")
                            for(int i = 0; i < qcList.size(); i++){
                                String qcId = qcList[i]
                                if(qcId.length() > 2){
                                    def di =  sh(script: "aws connect describe-queue --instance-id ${INSTANCEARN} --queue-id ${qcId}", returnStdout: true).trim()
                                    echo di
                                    def dq =  sh(script: "aws connect list-queue-quick-connects --instance-id ${INSTANCEARN} --queue-id ${qcId}", returnStdout: true).trim()
                                    echo dq
                                    def qc = jsonParse(di)
                                    def quickConnectList 
                                    try {
                                        quickConnectList = jsonParse(dq)
                                    } catch (Exception e) {
                                        echo "Exception occured while parsing QC " + e.toString()
                                    }                                    
                                    
                                    String targetQCList = ""
                                    String quickConnectConfig = ""
                                    if(quickConnectList) {
                                        echo "Collecting quick connect arns"
                                        for(int j=0; j< quickConnectList.QuickConnectSummaryList.size(); j++) {
                                            def obj = quickConnectList.QuickConnectSummaryList[j]
                                            String newId = getQuickConnectId(PRIMARYQC, obj.Name, TARGETQC)
                                            targetQCList = targetQCList.concat(" ").concat(newId)
                                        }
                                        echo "Here are collections for QC : ${targetQCList}"
                                        if(targetQCList.length() > 2) {
                                            quickConnectConfig = "--quick-connect ${targetQCList}"
                                        }
                                    }
                                    echo "QC config -> ${quickConnectConfig}"
                                    quickConnectList = null
                                    
                                    String qcName = qc.Queue.Name
                                    String qcDesc = qc.Queue.Description
                                    def ouboundFlowId 
                                    def qcCallerName
                                    if(qc.Queue.OutboundCallerConfig) {
                                        if(qc.Queue.OutboundCallerConfig.OutboundFlowId) {
                                            ouboundFlowId = getFlowId (PRIMARYCFS, qc.Queue.OutboundCallerConfig.OutboundFlowId, TARGETCFS)  
                                        }
                                        if(qc.Queue.OutboundCallerConfig.OutboundCallerIdName ) {
                                            qcCallerName = qc.Queue.OutboundCallerConfig.OutboundCallerIdName 
                                        }                                                                      
                                    }

                                    String hopId
                                    if(qc.Queue.HoursOfOperationId) {
                                        hopId = getHopId (PRIMARYHOP, qc.Queue.HoursOfOperationId, TARGETHOP)                                        
                                    }
                                    String maxContacts = ""
                                    if(qc.Queue.MaxContacts ) {
                                        maxContacts = " --max-contacts " + qc.Queue.MaxContacts 
                                    }
                                    String status = qc.Queue.Status 
                                    qc = null
                                    
                                    String outBoundConfig = "--outbound-caller-config \'"
                                    boolean nameEnabled = false
                                    boolean obConfigEnabled = false
                                    if(qcCallerName) {
                                        outBoundConfig = outBoundConfig.concat("OutboundCallerIdName=").concat(qcCallerName)
                                        nameEnabled = true
                                        obConfigEnabled = true
                                    }
                                    if(ouboundFlowId) {
                                        obConfigEnabled = true
                                        if(nameEnabled) {
                                            outBoundConfig = outBoundConfig.concat(",OutboundFlowId=").concat(ouboundFlowId)
                                        } else {
                                            outBoundConfig = outBoundConfig.concat("OutboundFlowId=").concat(ouboundFlowId)
                                        }
                                    }
                                    if(obConfigEnabled) {
                                        outBoundConfig = outBoundConfig.concat("\'")
                                    }
                                    def cq =  sh(script: "aws connect create-queue --instance-id ${TRAGETINSTANCEARN} --name ${qcName} --description \"${qcDesc}\" --hours-of-operation-id ${hopId} ${maxContacts} ${quickConnectConfig} ${outBoundConfig} " , returnStdout: true).trim()
                                    echo cq
                               }
                            }
                        }
                    }                
                }
            }
        } 

         stage('queue sync complete') {
            steps {
                echo "completed sycnronizing both instances"                
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {   
                    echo "completed sycnronizing both instances"                
                }
            } 
         }
        
     }
}


@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurper().parseText(json)
}

def toJSON(def json) {
    new groovy.json.JsonOutput().toJson(json)
}

def checkList(qcName, tl) {
    boolean qcFound = false
    for(int i = 0; i < tl.QueueSummaryList.size(); i++){
        def obj2 = tl.QueueSummaryList[i]
        String qcName2 = obj2.Name
        if(qcName2.equals(qcName)) {
            qcFound = true
            break
        }
    }
    return qcFound
}

def getFlowId (primary, flowId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for flowId : $flowId"
    for(int i = 0; i < pl.ContactFlowSummaryList.size(); i++){
        def obj = pl.ContactFlowSummaryList[i]    
        if (obj.Id.equals(flowId)) {
            fName = obj.Name
            println "Found flow name : $fName"
            break
        }
    }
    println "Searching for flow name : $fName"        
    for(int i = 0; i < tl.ContactFlowSummaryList.size(); i++){
        def obj = tl.ContactFlowSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found flow id : $rId"
            break
        }
    }
    return rId
}

def getQuickConnectId (primary, name, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = name
    String rId = ""
    echo "Find for name : ${fName}"       
    for(int i = 0; i < tl.QuickConnectSummaryList.size(); i++){
        def obj = tl.QuickConnectSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found id : $rId"
            break
        }
    }
    return rId
    
}

def getHopId (primary, userId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for userId : $userId"
    for(int i = 0; i < pl.HoursOfOperationSummaryList.size(); i++){
        def obj = pl.HoursOfOperationSummaryList[i]    
        if (obj.Id.equals(userId)) {
            fName = obj.Name
            println "Found name : $fName"
            break
        }
    }
    println "Searching for Id for : $fName"        
    for(int i = 0; i < tl.HoursOfOperationSummaryList.size(); i++){
        def obj = tl.HoursOfOperationSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found Id : $rId"
            break
        }
    }
    return rId
    
}

