# amazon-gamelift-anywhere-on-ecs-with-agent
Running Amazon GameLift anywhere fleets on Amazon ECS with GameLift Agent.

Using Amazon GameLift Anywhere fleets, you can run your game server processes on various computing platforms such as on-premises servers, EC2 instances, or container platforms such as Amazon ECS.

Amazon GameLift Agent is a Java application that is used to launch game server processes on Amazon GameLift fleets. 

Amazon GameLift Agent performs the following key tasks on Amazon GameLift fleets:

- Process management. Starts new game server processes, monitors server process status, and launches new server processes as existing processes end.
- Register the compute with an Amazon GameLift Anywhere fleet using RegisterCompute API.
- Calls GetComputeAuthToken API to fetch an authorization token and pases it to game server process as an environment variable.
- Periodically (default every 10 min) retrieves the latest fleet's runtime configuration from Amazon GameLift service and adjusts game server processes according to the updated runtime configuration.

Previously, it was previously only available on Amazon GameLift managed fleets. When running game servers on Amazon GameLift anywhere fleet, above tasks such as process management, registering computes, getting authorization token shall be implemented separately. 

Now Amazon GameLift Agent is available for use with Amazon GameLift anywhere fleets as it has been opened sourced at [Amazon GameLift Agent](https://github.com/aws/amazon-gamelift-agent) GitHub repo.

This post describe steps on running game servers on Amazon GameLift anywhere fleet with Amazon GameLift agent. Also the GameLift agent and game servers runs in a container on Amazon ECS tasks which will be register as computes to Amazon GameLift fleet. 



### 1. Dev Machine configuration

```
# GameLift Agent require Java 17 and maven version 3.2.5 or higher to compile.
# Install Java 17

sudo yum install git -y
sudo yum install java -y

# check installed java version
java -version
 openjdk version "17.0.12" 2024-07-16 LTS
 OpenJDK Runtime Environment Corretto-17.0.12.7.1 (build 17.0.12+7-LTS)
 OpenJDK 64-Bit Server VM Corretto-17.0.12.7.1 (build 17.0.12+7-LTS, mixed mode, sharing)

# install GameLift Agent
agent_file="amazon-gamelift-agent/target/GameLiftAgent-1.0.jar"

git clone https://github.com/aws/amazon-gamelift-agent.git
cd amazon-gamelift-agent/

# Install Maven
wget https://archive.apache.org/dist/maven/maven-3/3.2.5/binaries/apache-maven-3.2.5-bin.tar.gz
tar xvzf apache-maven-3.2.5-bin.tar.gz
ln -s apache-maven-3.2.5/ maven
MAVEN_HOME=$PWD/maven
PATH=$PATH:$MAVEN_HOME/bin

# Build GameLift Agent
mvn clean compile assembly:single
```



Note that to run GameLift Agent on Amazon ECS task by properly accessing ECS metadata services, following modification is applied.

https://github.com/hyundonk/amazon-gamelift-agent/commit/37d5d4b210ecda6fe1dcb21da5c3eb7182a8ae7d



### 2. Building game server with Amazon GameLift server SDK.

To-be-added



### 3. Build Docker container that runs GameLift Agent and game servers

```
# Write a Dockerfile
cat << EOF > Dockerfile
FROM unitymultiplay/linux-base-image:20240902

# Remove below
ENV AWS_CONFIG_FILE=/.aws/config

USER root

RUN mkdir -p /local/game/logs
RUN mkdir -p /local/game/agent

WORKDIR /local/game

COPY ./config $AWS_CONFIG_FILE

COPY ./sample.x86_64 /local/game/sample.x86_64
COPY ./sample_Data/ /local/game/sample_Data/
COPY ./UnityPlayer.so /local/game/UnityPlayer.so
COPY ./amazon-gamelift-agent/target/GameLiftAgent-1.0.jar /local/game/agent/

COPY ./start_game_session.sh /local/game/start_game_session.sh
RUN chmod +x /local/game/start_game_session.sh
RUN chmod +x /local/game/sample.x86_64

EXPOSE 7777

RUN apt-get -y update && apt-get -y install ca-certificates curl jq zip openjdk-17-jre
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
    unzip awscliv2.zip && \
	./aws/install

ENV PATH="$PATH:/user/local/bin"

#USER mpukgame

ENTRYPOINT ["./start_game_session.sh"]
EOF

# Get Auth Token for public repo
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/u7q3t7m6

# Build Docker image and upload to ECS repository
docker build -t unitygameserver --platform linux/amd64 .
docker tag unitygameserver:latest public.ecr.aws/u7q3t7m6/unitygameserver:latest
docker push public.ecr.aws/u7q3t7m6/unitygameserver:latest

```



### 4. Create Amazon ECS service and ECS task

```
AWS_REGION=ap-northeast-2
ACCOUNT_ID={MY_DEMO_ACCOUNT_ID}

ECS_CLUSTER_NAME={MY_DEMO_ECS_CLUSTER_NAME}
VPC_ID={MY_DEMO_VPC_ID}
PUBLIC_SUBNET1={MY_DEMO_PUBLIC_SUBNET1_ID}
PUBLIC_SUBNET2={MY_DEMO_PUBLIC_SUBNET2_ID}

# Create a ECS task role 
TASK_ROLE_NAME=ecs-game-server-task-role

cat << EOF > task-trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ecs-tasks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

TASK_ROLE_ARN=$(aws iam create-role --role-name $TASK_ROLE_NAME --assume-role-policy-document file://task-trust-policy.json --query 'Role.Arn' --output text)

cat << EOF > ecs-game-server-task-permission.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:DescribeLogGroups",
                "logs:CreateLogGroup"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:DescribeLogStreams",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:$AWS_REGION:$ACCOUNT_ID:log-group:/*"
            ]
        },
        {
            "Action": "gamelift:*",
            "Resource": "*",
            "Effect": "Allow"
        },
        { 
            "Effect": "Allow", 
            "Action": [ 
                "ec2:DescribeNetworkInterfaces",
                "ecs:DescribeTasks"
            ], 
            "Resource": "*" 
       }
    ]
}
EOF

aws iam put-role-policy --role-name $TASK_ROLE_NAME --policy-name 202409-ecs-game-server-task-permission --policy-document file://ecs-game-server-task-permission.json

# Create a ECS task execution role
TASK_EXECUTION_ROLE_NAME=ecs-game-server-task-execution-role

TASK_EXECUTION_ROLE_ARN=$(aws iam create-role --role-name $TASK_EXECUTION_ROLE_NAME --assume-role-policy-document file://task-trust-policy.json --query 'Role.Arn' --output text)

aws iam attach-role-policy --role-name $TASK_EXECUTION_ROLE_NAME --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy


# Create a ECS task definition
cat << EOF > game-server-task.json
{
    "family": "ecs-game-server-demo",
    "networkMode": "awsvpc",
    "executionRoleArn": "$TASK_EXECUTION_ROLE_ARN",
    "taskRoleArn": "$TASK_ROLE_ARN",
    "containerDefinitions": [
        {
          "essential": true,
          "image": "906394416424.dkr.ecr.us-west-2.amazonaws.com/aws-for-fluent-bit:latest",
          "name": "log_router",
          "firelensConfiguration": {
            "type": "fluentbit"
          },
          "logConfiguration": {
            "logDriver": "awslogs",
            "options": {
              "awslogs-group": "firelens-container",
              "awslogs-region": "$AWS_REGION",
              "awslogs-create-group": "true",
              "awslogs-stream-prefix": "firelens"
            }
          },
          "memoryReservation": 254
        },
        {
            "essential": true,
            "name": "gameserver",
            "image": "public.ecr.aws/u7q3t7m6/unitygameserver",
            "portMappings": [
              {
                "containerPort": 7777,
                "hostPort": 7777,
                "protocol": "udp"
              }
            ],
            "logConfiguration": {
                    "logDriver": "awsfirelens",
                    "options": {
                            "Name": "cloudwatch_logs",
                            "region": "$AWS_REGION",
                            "log_key": "log",
                            "log_group_name": "/aws/ecs/containerinsights/$ECS_CLUSTER_NAME/application",
                            "auto_create_group": "true",
                            "log_stream_name": "gameserver"
                    }
            },
            "dependsOn": [
            {
              "containerName": "log_router",
              "condition": "START"
            }
            ],
            "memoryReservation": 254
        }
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "cpu": "512",
    "memory": "1024"
}
EOF

aws ecs register-task-definition \
    --cli-input-json file://game-server-task.json \
    --region $AWS_REGION

```



### 5. Creating a GameLift fleet

```
aws gamelift create-fleet --name $FLEET_NAME --compute-type ANYWHERE \
             --locations "Location=$LOCATION_NAME" \
             --runtime-configuration "ServerProcesses="ServerProcesses=[{LaunchPath=/local/game/sample.x86_64,ConcurrentExecutions=1,Parameters=-logFile /local/game/logs/myserver1935.log -port 7777}]" \
             --anywhere-configuration Cost=0.2 \
             --query 'FleetAttributes.FleetId' --output text
             
```



### 6. Run ECS task

```
# Create security group for ECS tasks and add allow rules for HTTP
ECS_DEMO_SG=$(aws ec2 create-security-group --group-name ecs-game-server-demo-SG --description "ECS Game Sever Demo SG" --vpc-id $VPC_ID --region $AWS_REGION)

ECS_DEMO_SG=$(aws ec2 create-security-group --group-name ecs-game-server-demo-SG --description "ECS Game Sever Demo SG" --vpc-id $VPC_ID --region $AWS_REGION)

ECS_EXEC_DEMO_SG_ID=$(aws ec2 describe-security-groups --filters Name=group-name,Values=ecs-game-server-demo-SG --query 'SecurityGroups[*].GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $ECS_EXEC_DEMO_SG_ID --protocol udp --port 7777 --cidr 0.0.0.0/0 --region $AWS_REGION 

aws ecs run-task \
    --cluster $ECS_CLUSTER_NAME \
    --task-definition ecs-game-server-demo \
    --network-configuration awsvpcConfiguration="{subnets=[$PUBLIC_SUBNET1, $PUBLIC_SUBNET2],securityGroups=[$ECS_EXEC_DEMO_SG_ID],assignPublicIp=ENABLED}" \
    --overrides '{"containerOverrides":[{"name": "gameserver", "environment": [{"name": "LOCATION", "value": "custom-anywhere-location"}, {"name": "FLEET_ID", "value": "$FLEET_ID"}, {"name": "PORT", "value": "7777"}]}]}' \
    --enable-execute-command \
    --launch-type FARGATE \
    --platform-version '1.4.0' \
    --region $AWS_REGION
```



### 6. Check ECS task

```
aws ecs execute-command  \
    --region $AWS_REGION \
    --cluster ecs-game-server-demo \
    --task $ECS_TASK_ID \
    --container gameserver \
    --command "/bin/bash" \
    --interactive
```



### 7. Test with Unity Game Client

#### First run matchmaking using python script. This will give server address and port along with player session id. 

```
python3 ./Test/send-matchmaking.py
call start_matchmaking...
ticketID: 
matchmaking status:  PLACING
matchmaking status:  PLACING

Match created:  1.2.3.4 7777 {'PlayerId': '1', 'PlayerSessionId': 'psess-a9f4bdc4-1d49-fa37-d17b-fffffff477ae'}
Matchmaking succeeded. PlayerSessionId:  psess-a9f4bdc4-1d49-fa37-d17b-fffffff477ae
```

#### Then start Unity client with the player session Id and game server IP address as parameters.

```
./Builds/client/sample.app/Contents/MacOS/unity-gamelift-integration-sample -playerSessionId psess-a9f4bdc4-1d49-fa37-d17b-fffffff477ae -serverIp 1.2.3.4
```

