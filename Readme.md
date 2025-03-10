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

1. **Create Environment**: Initializes the environment required for testing Open5GS.
2. **Deploy New Version**: Deploys the latest version of the Open5GS PCF to the inbound registry.
3. **Deploy Simulators**: Deploys the required simulators that interact with Open5GS for testing.
4. **Test Execution Using Rebaca ABot**: Executes tests using the Rebaca ABot.
5. **Results Validation**: Validates the test results and ensures that all test cases pass successfully.
6. **Report Generation**: Generates a detailed test report, including success or failure details.
7. **Promote Version to Integration Registry**: If all test cases pass, the image is promoted from the inbound registry to the outbound registry for integration.
8. **Cleanup Environment**: Cleans up the testing environment.

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
