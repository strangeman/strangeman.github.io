---
layout: post
categories: devops
title: "CI/CD for Deploying Flows in Prefect Cloud 2"
description: "Some notes about tinkering with a deployment process for a Prefect Cloud"
keywords: "ci-cd, prefect"
---
# CI/CD for Deploying Flows in Prefect Cloud 2

## Initial Requirements

- We've got a bunch of Python scripts representing various Flows, all packed in one repository.
- The goal is to deploy the same Flow in the form of multiple Deployments (with different parameters and schedules).
- Also, we need to deploy the Flows in two different environments (Dev and Prod) for testing and debugging.
- We prefer defining everything in code rather than messing around with manual configurations in a web interface

## What Prefect Offers

The Prefect documentation ([https://docs.prefect.io/2.14.3/](https://docs.prefect.io/2.14.3/)) is a bit confusing, but after some trial and error, here's what I found out:

- Prefect gives us two methods to execute Flows - using agents and workers ([https://docs.prefect.io/2.14.3/guides/upgrade-guide-agents-to-workers/](https://docs.prefect.io/2.14.3/guides/upgrade-guide-agents-to-workers/)). The difference, as far as I can tell, lies in setting up the underlying infrastructure. With agents, you configure it in the infrastructure block ([https://docs.prefect.io/2.14.3/concepts/infrastructure/](https://docs.prefect.io/2.14.3/concepts/infrastructure/)), and for workers, it's done through job templates. We opted for agent-based deployments to avoid tweaking our Flows too much.
- There are three main ways to deploy our Deployments:
    1. `prefect deployment build` + `prefect deployment apply` - this lets you set parameters for each Deployment individually. It's an a bit more legacy command that supports both agent and worker configurations.
    2. `prefect deploy` - it can read the `prefect.yaml` config file and deploy everything from there, or you can specify parameters for a single deployment, similar to the previous method. This is a newer approach that only works with workers and doesn't support things like infrastructure blocks. Plus, it has a limitation - it can deploy either a single flow or all the flows listed in `prefect.yaml`, and you can't have multiple configs like that in one project.
    3. Call the `serve` or `to_deployment` functions directly from your Flow code.

## Description of Deployment Infrastructure

To keep things simple, we decided to skip storage blocks and store all Flows directly in the Docker image of the agent. This image is also used for continuous integration (CI):

```Dockerfile
# change to exact version if needed, e.g. 2.8.6-python3.10
# see https://hub.docker.com/r/prefecthq/prefect/tags?page=1
FROM prefecthq/prefect:2-python3.10

RUN /usr/local/bin/python -m pip install --upgrade pip

WORKDIR /opt/prefect

ENV PYTHONPATH "${PYTHONPATH}:/opt/prefect/"
ENV PREFECT_QUEUE "default"

COPY requirements.txt /opt/prefect/requirements.txt
RUN pip install -r requirements.txt
RUN prefect block register -m prefect_aws.ecs

COPY src /opt/prefect/src/
COPY flows /opt/prefect/flows/

# default CMD for agent
# workers will be spawned with their own cmd
CMD [ "sh", "-c", "prefect agent  start -q ${PREFECT_QUEUE}" ]
```

For Deployments, we chose the first deployment method - using `prefect deployment build`. We wanted to continue defining the infrastructure alongside the Flow in code (as infrastructure blocks). The method of "deploy everything from the project" didn't suit us because we needed to deploy flows in two different environments in different stages of development.

However, the idea of having a config file describing all deployments was quite handy. So, we came up with our own wrapper that reads our YAML config specifying deployments for a specific environment.

Here's the script we came up with:

```python
import yaml
import sys
import subprocess
import json

def read_yaml_file(file_path):
    try:
        with open(file_path, 'r') as yaml_file:
            data = yaml.safe_load(yaml_file)
            return data
    except FileNotFoundError:
        print(f"Error: File '{file_path}' not found.")
    except Exception as e:
        print(f"Error: {e}")

# construct_prefect_command accepts two parameters:
# deployment - dict with deployment parameters (see examples in prefect-deployments-dev.yaml)
# ib_block - infrastructure block with AWS creds and ECS config (located in the infrastructure folder)
def construct_prefect_command(deployment, ib_block):
    # mandatory fields
    if "entrypoint" not in deployment:
        raise KeyError("'entrypoint' field is mandatory")
    if "name" not in deployment:
        raise KeyError("'name' field is mandatory")

    command = f"prefect deployment build {deployment['entrypoint']} -n {deployment['name']} -a --skip-upload -ib {ib_block}"

    # optional fields
    if "parameters" in deployment:
        command += f" --params=\'{json.dumps(deployment['parameters'])}\'"

    if "version" in deployment:
        command += f" --version=\"{deployment['version']\""
    
    if "tags" in deployment:
        for tag in deployment["tags"]:
            command += f" --tag=\"{tag}\""

    if "schedule" in deployment:
        if "cron" in deployment["schedule"]:
            command += f" --cron=\"{deployment['schedule']['cron']}\""
        if "interval" in deployment["schedule"]:
            command += f" --interval=\"{deployment['schedule']['interval']}\""
        if "rrule" in deployment["schedule"]:
            command += f" --rrule=\"{deployment['schedule']['rrule']}\""
        if "timezone" in deployment["schedule"]:
            command += f" --timezone=\"{deployment['schedule']['timezone']}\""
        if "anchor-date" in deployment["schedule"]:
            command += f" --anchor-date=\"{deployment['schedule']['anchor-date']}\""

    if "work_pool" in deployment:
        if "name" in deployment["work_pool"]:
            command += f" --pool=\"{deployment['work_pool']['name']}\""
        if "work_queue_name" in deployment["work_pool"]:
            command += f" --work-queue=\"{deployment['work_pool']['work_queue_name']}\""

    return command

if __name__ == "__main__":

    if len(sys.argv) != 2:
        raise OSError("Script accepts exactly one argument: path to deployments YAML config")
    file_path = sys.argv[1]
    yaml_data = read_yaml_file(file_path)
    if yaml_data:
        if "ib-block" not in yaml_data:
            raise KeyError("'ib-block' field is mandatory")
        if "deployments" not in yaml_data:
            raise KeyError("'deployments' field is mandatory")

        for deployment in yaml_data["deployments"]:
            deploy_command = construct_prefect_command(deployment, yaml_data["ib-block"])
            print(f"run {deploy_command}")
            subprocess.run(deploy_command, shell=True, universal_newlines=True, check=True)
```

The logic is straightforward: parse the YAML file, check its structure, and convert its fields into parameters for the `prefect deployment build` command. The YAML file has the following structure:

```yaml
ib-block: "ecs-task/dev" # Mandatory field. Credentials and parameters for ECS from the infrastructure folder

deployments:
  - name: Flow1 # Mandatory field. Deployment name
    entrypoint: "examples/hello_world.py:hello" # Mandatory field. The path to the .py file containing the flow you want to deploy (relative to the root directory of your project) combined with the name of the flow function.
    parameters: # Optional field. Dict with parameters for flow
      number: 42
      name: "Do not panic!"
      env.stage: "dev"
    schedule: # Optional field. Scheduler parameters, "cron", "interval" and "rrule" are supported, see https://docs.prefect.io/2.14.3/concepts/schedules/?h=schedules#schedule-types
      cron: "0 0 * * *"
      timezone: "America/Chicago"
    tags: # Optional field. List of tags for deployment
      - "dev"
      - "some-tag1"
    version: "0.0.1" # Optional field. Deployment version
    work_pool: # Optional fields. Pool and Work Queue names
      name: "pool"
      work_queue_name: "dev"
  - name: Flow2
 <...>
```

All of this is deployed using Github Actions (some code parts are omitted as they are not relevant to the topic):

```yaml
name: Deploy to dev
on:
  workflow_dispatch:
  push:
    branches:
      - dev 

jobs:
  build-image:
    name: Build Prefect docker image
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.docker_image.outputs.IMAGE }}

    steps:
      - name: Check out code
        uses: actions/checkout@v3

      - name: Build, tag, and push docker image
        id: docker_image
        env:
          REGISTRY: XXXXX
          REPOSITORY: prefect2-worker
          IMAGE_TAG: dev
        run: |
          docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
          docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
          echo "IMAGE=${{ env.REGISTRY }}/${{ env.REPOSITORY }}:${{ env.IMAGE_TAG }}" >> $GITHUB_OUTPUT

  blocks:
    name: Prefect Blocks Upload
    runs-on: ubuntu-20.04 # need for glibc compatibility with prefect docker container
    needs: build-image

    container:
      image: "${{ needs.build-image.outputs.image }}" # run in prefect container from the first step

    env:
      PREFECT_API_KEY: ${{ secrets.PREFECT_API_KEY}}
      WORKSPACE_NAME: "example"
      PREFECT_IMAGE: "${{ needs.build-image.outputs.image }}"

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Authenticate to Prefect Cloud and upload block
        run: |
          prefect config set PREFECT_API_KEY=${{ secrets.PREFECT_API_KEY }} 
          prefect config set PREFECT_API_URL=${{ secrets.PREFECT_API_URL }}
          python3 infrastructure/ecs-task-dev.py # upload infrastructure block to prefect cloud

      - name: Blocks creation finished
        run: echo "ECS block built at $(date +'%Y-%m-%dT%H:%M:%S')" >> $GITHUB_STEP_SUMMARY

  deploy-flows:
    runs-on: ubuntu-20.04
    needs: [build-image, blocks]

    container:
      image: "${{ needs.build-image.outputs.image }}" # run in built prefect container from the first step

    steps:
      - uses: actions/checkout@v3

      - name: Authenticate to Prefect Cloud
        run: |
          prefect config set PREFECT_API_KEY=${{ secrets.PREFECT_API_KEY }} 
          prefect config set PREFECT_API_URL=${{ secrets.PREFECT_API_URL }}

      - name: Deploy flows to Cloud
        id: build
        run: |
          python upload_deployments.py prefect-deployments-dev.yaml
          echo "Flows from prefect-deployments-dev.yaml deployed to Prefect Cloud (dev)" >> $GITHUB_STEP_SUMMARY
```

## Conclusion

This solution enabled us to implement the necessary functionality without making any changes to the Flow code. Moreover, since the deployment script is written in Python and is very straightforward, we were able to hand it over to the Data team. This allows them to modify it in the future to better suit their needs.
