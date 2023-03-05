#!groovy
import groovy.json.JsonOutput
import groovy.json.JsonSlurper

def getWorkspaceId(organization, workspace_name) {
    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
        url: "https://${env.TFHOST}/api/v2/organizations/" + organization + "/workspaces/" + workspace_name
    )

    def data = new JsonSlurper().parseText(response.content)
    if (!data.data.id) {
        println 'cannot find workspace'
        exit 0
    } else {
        println 'found workspace'
        println ('Workspace Id: ' + data.data.id)
        return data.data.id
    }
}


def getWorkspaceVars(workspace_id, var_keys) {
    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        url: "https://${env.TFHOST}/api/v2/workspaces/" + workspace_id + "/vars"
    )

    Map data = new JsonSlurper().parseText(response.content)

    Map toreturn = [:]

    for (item in data.data) {
        if (var_keys.contains(item.attributes.key)) {
            toreturn[item.attributes.key] = item.id
        }
    }
    return toreturn
}

def updateWorkspaceVar(workspace_id, var_id, var_key, var_value) {
    println ('updateWorkspace Var key: ' + var_key + ' value:' + var_value)
    def payload = """
            {
                "data": {
                  "id":"${var_id}",
                  "attributes": {
                    "key":"${var_key}",
                    "value":"${var_value}",
                    "category":"terraform",
                    "hcl": false,
                    "sensitive": false
                  },
                  "type":"vars"
                }
            }
          """

    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'PATCH',
        requestBody: "${payload}",
        url: "https://${env.TFHOST}/api/v2/workspaces/" + workspace_id + "/vars/" + var_id
    )
}


def createConfigurationVersion(workspace_id) {
    def payload = '''
                  {
                      "data": {
                        "type": "configuration-versions",
                        "attributes": {
                          "auto-queue-runs": false
                        }
                      }
                  }
    '''

    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'POST',
        requestBody: "${payload}",
        url: "https://{env.TFHOST}/api/v2/workspaces/" + workspace_id + '/configuration-versions'
    )

    def data = new JsonSlurper().parseText(response.content)
    upload_url = data.data.attributes['upload-url']
    println ("upload url" + upload_url)
    cv_id = data.data.id
    println ("configuration-version id" + cv_id)
    return upload_url
/*
    # Upload configuration
echo "Uploading configuration version using ${config_dir}.tar.gz"
curl -s --header "Content-Type: application/octet-stream" --request PUT --data-binary @${config_dir}.tar.gz "$upload_url"
*/

}


def doRun(workspaceid) {
    def payload = """
{
    "data": {
        "attributes": {
            "is-destroy":false,
            "message": "Triggered run from Jenkins (build #${env.BUILD_TAG})"
        },
        "type":"runs",
        "relationships": {
            "workspace": {
                "data": {
                    "type": "workspaces",
                    "id": "$workspaceid"
                }
            }
        }
    }
}
    """
    // createRun
    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
        httpMode: 'POST',
        requestBody: "${payload}",
        url: "https://${env.TFHOST}/api/v2/runs"
    )
    def data = new JsonSlurper().parseText(response.content)

    println ("Run id: " + data.data.id)
    println ("Run status" + data.data.attributes.status)

    return data.data.id
}


def listLiveRun(workspaceid) {
    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
        url: "https://${env.TFHOST}/api/v2/workspaces/${workspaceid}/runs"
    )
    def data = new JsonSlurper().parseText(response.content)

    println ("Number of runs: " + data.meta.pagination.'total-count')
    def result =  data.data
    def live_runs = []
    def i = 0
    result.each {
        def status = it.attributes.status
        if (status == "discarded" || status == "applied" || status == "errored" || status == "canceled" || status == "force_canceled") {
            i = i + 1
        } else {
            println "$it.id : $it.attributes.status"
            live_runs.push(it.id)
        }
    }

    i = live_runs.size()
    println ("number of live runs:" + i)
    return live_runs
}

def listSentinelPolicysets (orgId) {
    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
        url: "https://${env.TFHOST}/api/v2/organizations/${orgId}/policy-sets"
    )
    def data = new JsonSlurper().parseText(response.content)
    
    policy_count = data.meta.pagination.'total-count'
    println ("Number of Sentinel policy sets: " + policy_count)

    return policy_count
}
//
def waitForPlan(runid) {
  def status =''
  println('waitForPlan - id: ' + runid)
  while (status!="errored") {
    status = getRunStatus(runid)
    println('Status: ' + status)
    // // If a policy requires an override, prompt in the pipeline
    // if (status == 'finished') {
    //   def getPlanResults = getPlanResults(runid)
    //   return getPlanResults
    // }
    switch (status) {
        case 'finished':
          def getPlanResults = getPlanResults(runid)
          return getPlanResults
        case 'errored':
          println "Plan failed"
          return 0
    }

    sleep(5)
  }
}


def discardRun(runid) {
    def result = ''

    println "discardingRun"
    println ("Run id: " + runid)

    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'POST',
        url: "https://${TFHOST}/api/v2/runs/${runid}/actions/discard"
    )

    println response
    //println ("Run id: " + data.data.id)
    //println ("Run status" + data.data.attributes.status)

    //def data = new JsonSlurper().parseText(response.content)
    //result = data.data.attributes.status
    //return result
}

/* 
def getPlanStatus(runid) {
    def result = ''

    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://${TFHOST}/api/v2/runs/${runid}/plan"
    )
    def data = new JsonSlurper().parseText(response.content)
    result = data.data.attributes.status
    return result
}
*/



def getPlanResults(runid) {
    def result = ''

    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://${TFHOST}/api/v2/runs/${runid}/plan"
    )
    def data = new JsonSlurper().parseText(response.content)
    def planResults = data.data.attributes."log-read-url"
    return planResults
}


def waitForPolicy(runid) {
  def status =''
  status = getRunStatus(runid)
  println('waitForPolicy- id: ' + runid)
  printlin('status ' + status)

  while (status == "policy_checking") {
    status = getRunStatus(runid)
    println('Status: ' + status)
    switch (status) {
        case 'policy_checked':
          return 0
        case 'policy_override':
          return 0
        case 'policy_soft_failed':
          return 0
    }
    sleep(5)
  }
}

def waitForRun(runid) {
    def count = 0
    def running = true
    println('waitForRun - id: ' + runid)
    while ( running) {
        def status = getRunStatus(runid)
        println('waitForRun - status: ' + status)

        //if ((status == 'planned') && (isConfirmable == true) && (override == "no") ) break
        if (status == 'planned') running = false
        if (status == 'applied') running = false
        if (status == "planned_and_finished") running = false
        if (status == "errored") running = false
        if (status == "discarded") running = false
        if (status == "canceled") running = false
        if (status == "force_canceled") running = false
        if (status == "discarded") running = false
        //if (status == 'cost_estimated') break

        // If a policy requires an override, prompt in the pipeline
        /*
        if (status.startsWith('approve_policy')) {
            def override
            try {
                override = input (message: 'Override policy?', 
                    ok: 'Continue', 
                    parameters: [ booleanParam( 
                        defaultValue: false, 
                        description: 'A policy restriction is enforced.Check the box to approve overriding the policy.',
                                      name: 'Override')
                                 ])
            } catch (err) {
                override = false
            }

            // If we're overriding, tell terraform. Otherwise, discard the run
            if (override == true) {
                println('Overriding!')
                def item = status.split(':')[1]
                println ('item is ' + item)
                def overridden = overridePolicy(item)
                if (!overridden) {
                    println('Could not override the policy')
                    discardRun(runid)
                    error('Could not override the Sentinel policy')
                    break
                } 
            } else {
                println('Rejecting!')
                discardRun(runid)
                error('The pipeline failed due to a Sentinel policy restriction.')
                break
            }
        }
        */

        /*
        // If we're ready to apply, prompt in the pipeline to do so
        if (status == 'apply_plan') {
            def apply
            try {
                apply = input (message: 'Confirm Apply', ok: 'Continue', 
                    parameters: [booleanParam(defaultValue: false, 
                    description: 'Would you like to continue to apply this run?', name: 'Apply')]) 
            } catch (err) { 
                apply = false 
            }

            // If we're going to apply, tell Terraform. Otherwise, discard the run
            if (apply == true) {
                println('Applying plan')
                applyRun(runid)
                break
            }
            else {
                println('Rejecting!')
                discardRun(runid)
                error('The pipeline failed due to a manual rejection of the plan.')
                break
            }
        }
        */

        if (count > 60) break
        count++
        sleep(5)
    } //while continue
}

def getRunStatus(runId) {
  def result = ''
  def status = ''

  def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://${TFHOST}/api/v2/runs/${runId}"
    )
  def data = new JsonSlurper().parseText(response.content)
  status = data.data.attributes.status
  println ('getRunStatus - status:' + status)
  /*
  switch (status) {
      case 'pending':
      case 'plan_queued':
      	result = 'pending'
      	break
      case 'planning':
      	result = 'planning'
      	break
      case 'planned':
      case 'planned_and_finished':
      	result = 'planned'
      	break
      case 'cost_estimating':
        result = "costing"
        break
      case 'cost_estimated':
      	result = 'cost_estimated'
      	break
      case 'policy_checking':
      	result = 'policy'
      	break
      case 'policy_override':
      	println(response.content)
      	result = 'approve_policy:' + data.data.relationships['policy-checks'].data[0].id
      	break
      case 'applied':
      	result = 'applied'
      	break
      case 'policy_checked':
      	result = 'apply_plan'
      	break
      default:
       	result = 'running'
      	break
  }
  */
  return result
}

def isAutoApply(runid) {
    def result = ''

    def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [[ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ]],
        url: "https://${TFHOST}/api/v2/runs/${runid}/plan"
    )
    def data = new JsonSlurper().parseText(response.content)
    result = data.data.attributes.auto-apply
    println ('isAutoApply: ' + result)
    if (result == "false")
        return false
    return true
}

/*
def waitForApply(runid) {
    def count = 0
    def status = getRunStatus(runid)
    def autoApply = isAutoApply(runid)
    println('waitForApply - id: ' + runid)
    println('waitForApply - status:' + status)
    println('waitForApply - autoApply:' + autoApply)
    if ((status == 'cost_estimated' || status == 'policy_checked') && !autoApply) {
        def payload = """
        {
            "comment": "automatically applied from Jenkins" 

        }
        """
        println "waitForApply - Apply from Jenkins"
        def response = httpRequest(ignoreSslErrors: true,
            customHeaders: [
                [ name: "Authorization", value: "Bearer " + env.BEARER_TOKEN ],
                [ name: "Content-Type", value: "application/vnd.api+json" ]
            ],
            httpMode: 'POST',
            requestBody: "${payload}",
            url: "https://${env.TFHOST}/api/v2/runs/${runid}/actions/apply"
        )
        def data = new JsonSlurper().parseText(response.content)

        println ("Run id: " + data.data.id)
        println ("Run status" + data.data.attributes.status)

    }
    while (true) {
        status = getRunStatus(runid)
        println('waitForApply-Status: ' + status)

        if (status == 'discarded') {
            println('This run has been discarded')
            error('The Terraform run has been discarded, and the pipeline cannot continue.')
            break
        }
        if (status == 'canceled') {
            println('This run has been canceled outside the pipeline')
            error('The Terraform run has been canceled outside the pipeline, and the pipeline cannot continue.')
            break
        }
        if (status == 'errored') {
            println('This run has encountered an error while applying')
            error('The Terraform run has encountered an error while applying, and the pipeline cannot continue.')
            break
        }
        if (status == 'applied') {
            println('This run has finished applying')
            break
        }

        if (count > 60) break
        count++
        sleep(5)
    }
}
*/


def overridePolicy(policyid) {
  def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'POST',
        url: "https://${TFHOST}/api/v2/policy-checks/${policyid}/actions/override"
    )

  def data = new JsonSlurper().parseText(response.content)
  if (data.data.attributes.status != 'overridden') {
    return false
  }
    else {
    return true
    }
}


def applyRun(runid) {
  def response = httpRequest(ignoreSslErrors: true,
        customHeaders: [
                [ name: 'Authorization', value: 'Bearer ' + env.BEARER_TOKEN ],
                [ name: 'Content-Type', value: 'application/vnd.api+json' ]
            ],
        httpMode: 'POST',
        responseBody: '{ comment: "Apply confirmed" }',
        url: "https://${TFHOST}/api/v2/runs/${runid}/actions/apply"
    )
}

// for Vault
def configuration = [vaultUrl: 'http://54.167.13.52:8200',
        vaultCredentialId: 'vault-token-root']

def secrets = [
        [path: 'secret1/innovation-lab', engineVersion: 1, secretValues: [
                [envVar: 'api_token', vaultKey: 'api_token2']]
        ]
]

pipeline {
  agent any
  parameters {
      string(name: 'ORGANIZATION', defaultValue: 'innovation-lab', description: '')
      choice(
       	name: 'VM_TYPE',
        choices: ['t2.nano', 't2.micro', 't2.small', 't2.medium']
      )
      choice(
       	name: 'SERVICE_CATALOG_ITEM',
        choices: ['vm-instance', 'web-instance', 'db-instance']
      )
  }
  environment {
          TFHOST            = "app.terraform.io"
          VAULT_HOST        = "54.167.13.52"
          TF_WORKSPACE_NAME = "${params.SERVICE_CATALOG_ITEM}"
          TF_ORG_NAME       = "${params.ORGANIZATION}"
    }

    stages {
        stage('Get Token from Vault') {
            steps {
                script {
                    // created imperatively, so we can modified & used at later stages
                    env.BEARER_TOKEN = "notatoken"
                }
                script {
                    withVault([configuration: configuration, vaultSecrets: secrets]) {
                        env.BEARER_TOKEN = env.api_token
                    }
                }
                script {
                    echo "BEARER_TOKEN=${env.BEARER_TOKEN}"
                }
            }
        }
        
        
        stage('Get Workspace Id') {
            steps {
                script {
                    env.TF_WORKSPACE_ID =  getWorkspaceId(env.TF_ORG_NAME, env.TF_WORKSPACE_NAME)
                }
                script {
                    echo "TF_ORG_NAME is ${env.TF_ORG_NAME}"
                    echo "TF_WORKSPACE_NAME is ${env.TF_WORKSPACE_NAME}"
                    echo "TF_WORKSPACE_ID is ${env.TF_WORKSPACE_ID}"
                }
            }
        }

        stage('ListRuns in a Workspace') {
           steps{
                 script {
                     listLiveRun(env.TF_WORKSPACE_ID)
                 }
             }
        }

        stage('Modify Workspace Parameters') {
            steps {
		        script {
                    env.TF_WORKSPACE_ID =  getWorkspaceId(env.TF_ORG_NAME, env.TF_WORKSPACE_NAME)
                }

                script {
                    TFE_VARS = getWorkspaceVars(env.TF_WORKSPACE_ID, ['vm_type'])
                    println TFE_VARS
                }

                script {
                    updateWorkspaceVar(env.TF_WORKSPACE_ID, TFE_VARS.get('vm_type'), 'vm_type', params.VM_TYPE)
                }
            }
        }

        stage('WorkspaceStartRun') {
            steps{
                script {
                    PLAN_ID = doRun(env.TF_WORKSPACE_ID)
                }
            }
        }
        stage ('WorkspaceRun') {
            steps {
                script {
                    waitForRun(PLAN_ID)
                }
            }
        }
/*
        stage('Workspace-PlanStage') {
            steps {
                script {
                    waitForPlan(PLAN_ID)
                }
            }
        }
        
        stage('Workspace-PolicyStage') {
            steps {
                script {
                    //waitForRun(PLAN_ID)
                    waitForPolicy(PLAN_ID)
                }
            }
        }

        stage('Workspace-ApplyStage') {
            steps {
                script {
                    waitForApply(PLAN_ID)
                }
            }
        }
*/
    } //stage
} //pipeline
