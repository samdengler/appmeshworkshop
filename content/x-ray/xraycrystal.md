---
title: "Enable X-Ray for crystal"
date: 2018-09-18T16:01:14-05:00
weight: 10
---

The AWS X-Ray daemon is a software application that listens for traffic on UDP port 2000, gathers raw segment data, and relays it to the AWS X-Ray API. The daemon works in conjunction with the AWS X-Ray SDKs and must be running so that data sent by the SDKs can reach the X-Ray service.

* Register a new task definition to add the x-ray container, and enable tracing for the envoy container.

```bash
# Define variables #
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1)
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
  | jq ' .taskDefinition' \
  | jq ' .containerDefinitions |= map(
        if .name == "envoy" then .environment +=
          [
            {
              "name": "ENABLE_ENVOY_XRAY_TRACING",
              "value": "1"
            }
          ]
        else . end) ' \
  | jq ' .containerDefinitions +=
        [
          {
            "image": "amazon/aws-xray-daemon",
            "essential": true,
            "name": "xray",
            "portMappings": [
              {
                "hostPort": 2000,
                "protocol": "udp",
                "containerPort": 2000
              }
            ],
            "healthCheck": {
              "retries": 3,
              "command": [
                "CMD-SHELL",
                "timeout 1 /bin/bash -c \"</dev/udp/localhost/2000\""
              ],
              "timeout": 2,
              "interval": 5,
              "startPeriod": 10
            }
          }
        ]' \
  | jq ' del(.status, .compatibilities, .taskDefinitionArn, .requiresAttributes, .revision) '
); \
TASK_DEF_FAMILY=$(echo $TASK_DEF_ARN | cut -d"/" -f2 | cut -d":" -f1);
echo $TASK_DEF_NEW > /tmp/$TASK_DEF_FAMILY.json &&
# Register ecs task definition #
aws ecs register-task-definition \
  --cli-input-json file:///tmp/$TASK_DEF_FAMILY.json
```

* Update the service.

```bash
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1)
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service crystal-service-lb-blue \
  --task-definition "$(echo $TASK_DEF_ARN)"
```

* Wait for the service tasks to be in a running state.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1);
# Get task state #
_list_tasks() {
   aws ecs list-tasks \
     --cluster $CLUSTER_NAME \
     --service crystal-service-lb-blue | \
   jq -r ' .taskArns | @text' | \
     while read taskArns; do
       aws ecs describe-tasks --cluster $CLUSTER_NAME --tasks $taskArns;
     done | \
   jq -r --arg TASK_DEF_ARN $TASK_DEF_ARN \
     ' [.tasks[] | select( (.taskDefinitionArn == $TASK_DEF_ARN)
                     and (.lastStatus == "RUNNING" ))] | length'
}
until [ $(_list_tasks) == "3" ]; do
  echo "Tasks are starting ..."
  sleep 10s
  if [ $(_list_tasks) == "3" ]; then
    echo "Tasks started"
    break
  fi
done
```

Access the ECS console and find the latest revision of your crystal task definition. Notice there are 3 containers declared, one for the crystal microservice, one for the envoy proxy and a third one for the X-Ray deamon.

Now access the AWS Admin console and go to the X-Ray service. Once in the X-Ray section of the console, select **Service Map** from the left hand side menu. Wait a few seconds for the service map to render.

When you select a node or edge on an AWS X-Ray service map, the X-Ray console shows a latency distribution histogram. Latency is the amount of time between when a request starts and when it completes. A histogram shows a distribution of latencies. It shows duration on the x-axis, and the percentage of requests that match each duration on the y-axis.
