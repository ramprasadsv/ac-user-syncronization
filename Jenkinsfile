
import groovy.json.JsonSlurper
import groovy.json.JsonOutput; 

def CONFIGDETAILS 
String MISSINGLIST = ""
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
                      sh(script: "rm -r ac-user-syncronization", returnStdout: true)    
                   }catch (Exception e) {
                       echo 'Exception occurred: ' + e.toString()
                   }                   
                   sh(script: "git clone https://github.com/ramprasadsv/ac-user-syncronization.git", returnStdout: true)
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
        
        stage('Find missing users') {
            steps {
                script {
                    echo "Find missing users in the target instance"
                    def pl = jsonParse(PRIMARYUSERS)
                    def tl = jsonParse(TARGETUSERS)
                    int listSize = pl.UserSummaryList.size() 
                    println "Primary list size $listSize"
                    for(int i = 0; i < listSize; i++){
                        def obj = pl.UserSummaryList[i]
                        String qcName = obj.Username
                        String qcId = obj.Id
                        boolean qcFound = checkList(qcName, tl)
                        if(qcFound == false) {
                            println "Missing Name : $qcName Id : $qcId"                                                              
                            MISSINGLIST = MISSINGLIST.concat(qcId).concat(",")                                
                        }
                    }
                }
                echo "Missing list in the target instance -> ${MISSINGLIST}"
            }
        }
        
        stage('Create the missing users') {
            steps {
                echo "Create the missing users in the target instance "                
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {   
                    script {
                        if(MISSINGQC.length() > 1 ){
                            def qcList = MISSINGQC.split(",")
                            for(int i = 0; i < qcList.size(); i++){
                                String qcId = qcList[i]
                                if(qcId.length() > 2){
                                    def di =  sh(script: "aws connect describe-user --instance-id ${INSTANCEARN} --user-id ${qcId}", returnStdout: true).trim()
                                    echo di
                                    def user = jsonParse(di)
                                    
                                    String userName = " --username " + user.Username
                                    String pwd = "--password Connect@12312"                                    
                                    String firstName = user.IdentityInfo.FirstName
                                    String lastName = user.IdentityInfo.LastName
                                    String email = user.IdentityInfo.Email                                    
                                    String idInfo = " --identity-info " + "FirstName=" + firstName + ",LastName=" + lastName +",Email=" + email                                    
                                    String phoneType = user.PhoneConfig.PhoneType
                                    String autoAccept = user.PhoneConfig.AutoAccept
                                    String acw = user.PhoneConfig.AfterContactWorkTimeLimit
                                    String dpn = user.PhoneConfig.DeskPhoneNumber                                    
                                    String pc = " --phone-config " + "PhoneType=" + phoneType + ",AutoAccept=" + autoAccept + ",AfterContactWorkTimeLimit=" + acw + ",DeskPhoneNumber=" + dpn                                     
                                    String rpId = getRPId(PRIMARYRPS, user.PhoneConfig.RoutingProfileId, TARGETRPS)                                    
                                    String rp = "--routing-profile-id ${rpId}"                                    
                                    String spIds = "--security-profile-ids " + getSPIds(PRIMARYSECPROS, user.PhoneConfig.SecurityProfileIds, TARGETSECPROS)
                                    String hid = getHrRchyId(PRIMARYHRCHY, user.HierarchyGroupId, TARGETHRCHY)
                                    String hgi = ""
                                    if(hid.length() > 1 ) {
                                        hgi = "--hierarchy-group-id " + hid
                                    }
                                    di = null
                                    
                                    def cq =  sh(script: "aws connect create-user --instance-id ${TRAGETINSTANCEARN} ${userName} ${pwd} ${idInfo} ${spIds} ${rpId} ${hgi}" , returnStdout: true).trim()
                                    echo cq
                               }
                            }
                        }
                    }                
                }
            }
        } 

         stage('user sync complete') {
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
    for(int i = 0; i < tl.UserSummaryList.size(); i++){
        def obj2 = tl.UserSummaryList[i]
        String qcName2 = obj2.Username
        if(qcName2.equals(qcName)) {
            qcFound = true
            break
        }
    }
    return qcFound
}

def getRPId (primary, searchId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for Id : $searchId"
    for(int i = 0; i < pl.RoutingProfileSummaryList.size(); i++){
        def obj = pl.RoutingProfileSummaryList[i]    
        if (obj.Id.equals(searchId)) {
            fName = obj.Name
            println "Found Name : $fName"
            break
        }
    }
    println "Searching for name : $fName"        
    for(int i = 0; i < tl.RoutingProfileSummaryList.size(); i++){
        def obj = tl.RoutingProfileSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found matching id : $rId"
            break
        }
    }
    return rId
}

def getSPIds (primary, searchIds, target) {
    String rId = ""
    searchIds.each{ sId => 
        def pl = jsonParse(primary)
        def tl = jsonParse(target)
        String fName = ""
        println "Searching for Id : $sId"
        for(int i = 0; i < pl.SecurityProfileSummaryList.size(); i++){
            def obj = pl.SecurityProfileSummaryList[i]    
            if (obj.Id.equals(sId)) {
                fName = obj.Name
                println "Found Name : $fName"
                break
            }
        }
        println "Searching for name : $fName"        
        for(int i = 0; i < tl.SecurityProfileSummaryList.size(); i++){
            def obj = tl.SecurityProfileSummaryList[i]    
            if (obj.Name.equals(fName)) {
                rId = rId + " " + obj.Id
                println "Found matching id : $rId"
                break
            }
        }
    };
                  
    return rId
}

def getHrRchyId (primary, searchId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for Id : $searchId"
    for(int i = 0; i < pl.UserHierarchyGroupSummaryList.size(); i++){
        def obj = pl.UserHierarchyGroupSummaryList[i]    
        if (obj.Id.equals(searchId)) {
            fName = obj.Name
            println "Found Name : $fName"
            break
        }
    }
    println "Searching for name : $fName"        
    for(int i = 0; i < tl.UserHierarchyGroupSummaryList.size(); i++){
        def obj = tl.UserHierarchyGroupSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Id
            println "Found matching id : $rId"
            break
        }
    }
    return rId
}
