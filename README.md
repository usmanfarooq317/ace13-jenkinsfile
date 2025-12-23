# ACE Jenkins Pipeline

This repository contains a Jenkins pipeline (`Jenkinsfile`) that starts or restarts an IBM ACE container, ensures a broker and execution group (server) exist, and deploys a packaged BAR artifact (`BVSRegFix2.bar`).

**Contents**
- **`Jenkinsfile`**: Declarative Jenkins pipeline used by Jenkins (Multibranch Pipeline or Pipeline from SCM).
- **`BVSRegFix2.bar`**: Packaged BAR artifact (application bundle) intended to be deployed into the ACE environment inside the container.

**Quick Overview**
- **Pipeline stages**: `Run ACE Container` and `Deploy BAR File`.
- **Runtime**: The pipeline operates by calling Docker to run an ACE image, then uses ACE command-line tools (mqsicreatebroker, mqsistart, mqsideploy, etc.) inside the running container to manage the broker/server and deploy the BAR file.

Usage

- **Run in Jenkins**: Create a Pipeline or Multibranch Pipeline job and point it to this repository. Jenkins will load the `Jenkinsfile` and execute the defined stages.
- **Run manually (basic)**: The `Jenkinsfile` uses these commands. You can run them locally (adjust variables first):

```bash
# Run container (example uses values from Jenkinsfile)
docker run -d --name aceserver \
  -p 7600:7600 -p 7800:7800 -p 7843:7843 \
  --env LICENSE=accept moiz3388/ace13:latest tail -f /dev/null

# Copy BAR file into container
docker cp BVSRegFix2.bar aceserver:/tmp/BVSRegFix2.bar

# Deploy inside container (runs ACE profile then deploys)
docker exec aceserver bash -l -c '. /opt/ibm/ace-13/server/bin/mqsiprofile; mqsideploy PROD -e prodserver -a /tmp/BVSRegFix2.bar -w 60'
```

Configuration

- **Environment variables**: The `Jenkinsfile` defines several variables at the top which you can change to suit your environment:
  - **`IMAGE_NAME`**: Docker image used for the ACE runtime.
  - **`CONTAINER_NAME`**: Name of the Docker container to create or restart.
  - **`BROKER_NAME`** / **`SERVER_NAME`**: ACE broker and execution group names used for deployment.
  - **`ACE_PROFILE`**: Path to the ACE profile inside the container.
  - **`BAR_FILE`**: Name of the BAR artifact to deploy.

Troubleshooting & Notes

- **Docker access**: Jenkins must have permission to run Docker on the agent executing this pipeline (either run Jenkins agent on the same host as Docker or use a privileged agent).
- **Image availability**: Ensure `IMAGE_NAME` exists and is pullable by the Jenkins agent (private registry credentials may be required).
- **License acceptance**: The ACE image is run with `--env LICENSE=accept`; verify you are compliant with IBM licensing before use.
- **Ports**: The pipeline maps ports `7600`, `7800`, and `7843` — make sure these do not conflict with other services.
- **BAR file**: `BVSRegFix2.bar` should be present in the repository root or reachable in the workspace when the pipeline runs.


