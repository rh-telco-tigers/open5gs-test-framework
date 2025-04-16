# Open5GS Testing Automation Framework with OpenShift Pipeline

This repository contains an automation framework for testing Open5GS using an OpenShift pipeline. The framework automates the process of testing Open5GS versions, including deployment, test execution, results validation, and image promotion.

## Repository Structure

- **Playbook**: Contains the Ansible playbooks used for configuring and managing Open5GS deployment.
- **Routes**: Stores route configurations required for the pipeline.
- **Tasks**: Contains individual pipeline tasks.
- **Triggers**: Contains scripts or configurations related to triggering pipeline executions.
- **pipeline.yml**: Defines the overall pipeline workflow.
- **pvc.yml**: Defines Persistent Volume Claims (PVCs) used for storing and persisting data during the pipeline execution.

## Pipeline Workflow
The pipeline consists of eight tasks. What each task does is detailed below:

1. **Create Environment Task**  
   The create-environment task can be found under Tasks -> create-environment.yaml. This task is responsible for cloning the necessary repositories, including the one containing    playbooks for calling Rebaca Abot. These repositories are stored in workspaces, making them accessible to subsequent tasks in the pipeline.

2. **Deploy New Version Task**  
   This Tekton task is designed to deploy the Open5GS System Under Test (SUT) using Helm and retrieve the PCF pod IP addresses for subsequent tasks.  
   The image tag is retrieved dynamically from the webhook payload when the image is pushed to Quay.  

   The Tekton task runs a shell script that performs the following steps:  

   - **Helm Upgrade Command:**  
     The script navigates to the `/mnt/open5gs` directory and executes a Helm upgrade command to deploy the Open5GS SUT.  
     The image tag is dynamically passed to ensure the correct version is deployed.  

   - **Sleep for Stability:**  
     A 30-second pause is added to ensure the pods have enough time to start and stabilize before checking their status.  

   - **Store IP Addresses:**  
     - The script removes any existing `pcf_ip.txt` files to avoid stale data.  
     - It then retrieves the IP address of the `open5gs-sut-pcf-*` pod that is in the `Running` state, storing the result in `/mnt/pcf_ip.txt`.  

3. **Deploy-Simulator Task**  
   The Tekton task runs a shell script that performs the following steps:  

   - **Create Output Directory:**  
     It creates a `/mnt/output` directory to store the results.  

   - **Fetch AAP Job Template:**  
     Queries AAP to retrieve the Job Template ID for the template named Deploy_Simulators. 

   - **Trigger AAP Job:**  
      - Launches the AAP job using the fetched template ID.
      - Passes pcf_ip, nrf_ip, and TAG as extra variables to the job.

   - **Monitor Job Status:**  
     Continuously polls the AAP job endpoint until the job status is successful, failed, or canceled.

   - **Extract and Save Job Output:**
      - Captures job logs (stdout) and extracts:
          - User Token → saved to /mnt/output/user_token.txt
          - Configuration Update Response → saved to /mnt/output/update_config.json


4. **Test Execution Task**  
   The Tekton task runs a shell script that performs the following steps:  

   - **Create Output Directory:**  
     It creates a `/mnt/output` directory to store the results.  

   - **Run the Ansible Job Template:**  
     The task runs the `Execute_Test` Job Template, which performs following steps:

     - **User Authentication:**  
       - Reads the token from the file to ensure secure access for further API requests.  

     - **Execute Feature File:**  
       - The playbook sends a POST request to (https://{{ abot_endpoint }}/abotrest/abot/api/v5/feature_files/execute) with the user_token as a Bearer token for authorization.
       - It sends the pcf-as-sut as a parameter in the request body.
       - The request only proceeds if the update_config_response.status is 200 (indicating that the configuration update was successful).
       - The response of the feature file execution is stored in the featurefile_response variable.

       
    
5. **Results Validation Task**:
   The Tekton task runs a shell script that performs the following steps:  

   - **Create Output Directory:**  
     It creates a `/mnt/output` directory to store the results.  

   - **Fetch AAP Job Template:**  
     - API Called: GET /api/v2/job_templates/?name=Result_Validation
     - Queries AAP to retrieve the Job Template ID for the template named Result_Validation.

   - **Trigger AAP Job:**  
      - API Called: POST /api/v2/job_templates/<JOB_TEMPLATE_ID>/launch/
      - Launches the AAP job using the retrieved Job Template ID.
      - Passes user_token (read from /mnt/output/user_token.txt) as an extra variable.
   
   - **Monitor Job Status:**  
     Continuously polls the AAP job endpoint until the job status is successful, failed, or canceled.

   - **Extract and Save Job Output:**
      - API Called: GET /api/v2/jobs/<LATEST_JOB_ID>/stdout/?format=txt
      - Captures job logs (stdout) and extracts:
           - Latest artifact timestamp → saved to /mnt/output/latest_artifact.txt
         


5. **Report to Slack**:
   The Tekton task runs a shell script that performs the following steps:  

   - **Create Output Directory:**  
     It creates a `/mnt/output` directory to store the results.

   - **Run the Ansible Playbook:**  
     The task runs the `send_report.yml` playbook, which performs following steps:

       - **Get Execution Summaryn:**  
         - The playbook makes a GET request to an API endpoint to retrieve the execution summary. The results are saved in a variable for the next steps.
       - **Check Failures:**  
         - The playbook examines the retrieved data to find any failed tests. If failures are detected, it prepares a Slack message listing the failed features. If all tests pass, the message confirms that the image is ready for promotion to the integration registry.
       - **Send Slack Message:**  
         - Depending on the test results, a Slack message is sent to notify the team. This helps everyone stay updated without manual checks.
       - **Save Final Result:**  
         - The outcome (TRUE or FALSE) is saved in a text file for reference in the tekton task.


6. **Promote New Version Task**:
   This tekton task automates the process of promoting the image that has passed all test cases. It takes the image tag as a parameter and transfers the image from inbound repositroty to integration repository. 
   
7. **Destroy Task**:
   This task deletes the /mnt/ansible-repo, /mnt/output, and /mnt/open5gs directories to ensure a clean state for the next pipeline run.
## Triggering the Pipeline

Once a new version of the Open5GS PCF is pushed to the inbound registry, it triggers the pipeline. The pipeline proceeds with the following steps:
- The new version is deployed.
- Tests are executed using Rebaca ABot.
- Test results are validated, and reports are generated and sent to Slack for notifications.
- If the image passes all tests, it is promoted to the integration registry.

## Usage

To trigger the pipeline, follow these steps:
1. Push the new Open5GS PCF image to the inbound registry.
2. The pipeline will be automatically triggered.
3. Monitor the test results and notifications in Slack.
4. Upon successful test execution, the image will be promoted to the integration registry.

## Requirements

- OpenShift 4.15 or above
- Ansible
- Rebaca ABot for test execution
- Slack integration for notifications
