# `redhat-codeready-dependency-analysis`
**Please Note: this Task is only compatible with Tekton Pipelines versions 0.19.0 and greater!**

## Overview
The redhat-codeready-dependency-analysis task is an interface between Tekton and Red Hat CodeReady Dependency Analytics platform. 
It provides vulnerability and compliance analysis for your applications dependencies, along with recommendations to address security vulnerabilities and licensing issues.

This task reflects [Dependency Analytics VS Code plugin](https://marketplace.visualstudio.com/items?itemName=redhat.fabric8-analytics) for Tekton Pipelines.

**Please Note: Currently this Task supports Golang, NPM, Python and JAVA/MAVEN ecosystems, support for other ecosystems will be provided very soon.**

## Prerequisite

#### 1. Creation of Workspace
Workspace is used as a common filesystem between tasks and used for inputs and outputs of task, in workspace dependencies are installed for further usage while running the task.

This [sample](../0.3/samples/workspace.yaml) can be referred to create a workspace.<br />

#### 2. Manifest File
This task scans manifest file (ex. requirements.txt, requirements-dev.txt etc.) for collecting a list of dependencies, hence prior to this task, target manifest file must be placed by user into workspace. 
The path to manifest is passed as parameters to task.

#### 3. Dependency Installation
Users need to install dependencies from manifest into a directory of workspace. 
Location of installation directory in workspace is required as parameter in the task.

#### 4. Adding Secret
This task requires two secrets, `crda_key` and `consent_telemetry` to be added in environment.

To authenticate a user this task uses CRDA user key. 
This key should be saved in secrets, but before that users need to generate it.<br />
This is a one time activity to perform. 

Here are the steps to generate a CRDA user key.<br />
1. Install CRDA CLI from [here](https://github.com/fabric8-analytics/cli-tools/releases).
2. Run `crda auth` command, and it will assign user a unique id, which can be found in `~/.crda/config.yaml`. 
Users need to store the value of `crda_key` into a `Secret` to run the task.

Second secret required is `consent_telemetry`. To improve CRDA platform and provide a better user experience, this task collects anonymous usage data, but only after user's approval. 
Please go thorough our [privacy statement](https://developers.redhat.com/article/tool-data-collection) to learn more about it. To opt-in for data collection users need to set `consent_telemetry=true` else set `consent_telemetry=false` to opt-out.
<br />
This [sample](../0.3/samples/secret.yaml) can be referred to create a secret file, replace `{{ CRDA_USER_KEY }}` with the generated CRDA key before running.
<br />
Following command can be used to create workspace from the file.

```
kubectl apply -f secret.yaml -n <NAMESPACE>
```

## Task Parameters
- **manifest-file-path**: Path of target manifest file to perform analysis. `(default: requirements.txt)`
- **output-file-name**: Path of the file to save analysis report. `(default: redhat-codeready-dependency-analysis-report.json)`
- **pkg-installation-directory**: Path of a directory in workspace, where dependencies are installed. `(default: site-package)`
- **image**: Image where CRDA CLI binary and required applications are installed. `(default: quay.io/ksoneji/crda:python)`. <br />List of images for different ecosystem and versions can be found [here](https://github.com/fabric8-analytics/crda-images/blob/main/README.md)
- **CRDA_HOST_URL**: CRDA host, adjust this if you are using an on premise CRDA environment. `(default: "")`

## Sample Output

Result of the task is saved as JSON format in the workspace directory having manifest file. 
```
{
  "report": {
    "total_scanned_dependencies": 20,
    "total_scanned_transitives": 58,
    "total_vulnerabilities": 15,
    "direct_vulnerable_dependencies": 8,
    "publicly_available_vulnerabilities": 10,
    "vulnerabilities_unique_to_snyk": 5,
    "critical_vulnerabilities": 3,
    "high_vulnerabilities": 3,
    "medium_vulnerabilities": 4,
    "low_vulnerabilities": 5,
    "report_link": "https://recommender.api.openshift.io/api/v2/stack-report/bc52c6b406f646bc84ed24fb24f5bdc9"
  },
  "exit_code": 2
}

```
This JSON data is having details about vulnerabilities and a link to view a detailed report is also provided. This JSON can be used by next task for making a decision to Fail/Pass the build.  

In the logs, a simplified report is shown, below is a sample report given in log.

```
=============================================
RedHat CodeReady Dependency Analysis Report
=============================================
Total Scanned Dependencies            :  6 
Total Scanned Transitive Dependencies :  38 
Total Vulnerabilities                 :  6 
Direct Vulnerable Dependencies        :  3 
Publicly Available Vulnerabilities    :  6 
Vulnerabilities Unique to Snyk        :  0 
Critical Vulnerabilities              :  1 
High Vulnerabilities                  :  5 
Medium Vulnerabilities                :  0 
Low Vulnerabilities                   :  0 
=============================================

Open this link to see detailed report:
https://recommender.api.openshift.io/api/v2/stack-report/bc52c6b406f646bc84ed24fb24f5bdc9 

Report is saved into file: <workspace>/redhat-codeready-dependency-analysis-report.json
Task is completed.
```

The link to detailed report will take users to a browser window having similar format as [Dependency Analytics VS Code plugin](https://marketplace.visualstudio.com/items?itemName=redhat.fabric8-analytics). <br /> To view premium vulnerability data, users can register with Snyk token.

![Alt Text](https://raw.githubusercontent.com/fabric8-analytics/fabric8-analytics-vscode-extension/master/images/0.3.0/reg-stack-analysis.gif)

## Install the CodeReady Dependency Analysis Task
```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/redhat-codeready-dependency-analysis/0.3/redhat-codeready-dependency-analysis.yaml -n <NAMESPACE>
```
## Install Git Clone Task
```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.3/git-clone.yaml -n <NAMESPACE>
```

## Usage

This task expects a secret named `crda` to exist with a valid CRDA user key in `crda-key`, an attached workspace having target manifest file and its dependencies installed in a directory.

The following PipelineRun and TaskRun demonstrate usage of the  Task. Please configure required components like workspace, secrets, tasks from YAMLs from `samples` directory:

- [Usage of redhat-codeready-dependency-analysis task in an end to end pipeline. (configure secret, workspace, git-clone, pre-task-setup, post-task-setup and pipeline from YAML provided in samples)](../0.3/samples/pipeline-run.yaml)
- [Running stand alone redhat-codeready-dependency-analysis task, assuming manifest file is already placed in workspace and dependencies are installed in a directory of workspace.](../0.3/samples/task-run.yaml)
