# MC-AWS-SERVER

A set of cloud formation stacks to quickly spin up a Minecraft Server.

## Features

- EBS Volume for Persistent Storage
- Elastic IP to maintain a Static IP Address
    - To-Do: Separate EIP Stack from EC2 Stack to allow EC2 stack to be deleted completely without changing the EIP.
- Utilizes itzg/minecraft-server
- Whitelist your IP Address to block public access
- Specify Availability Zone
- Specify Instance Type
- Specify Volume Size
- Specify Minecraft World Seed


## Useage

### Setup .env file

To the root, add a `.env` file with the following:

```
# Change values as desired to customize.

EBS_STACK_NAME=[placeholder]
EC2_STACK_NAME=[placeholder]

VOLUME_SIZE=[placeholder]
INSTANCE_SIZE=[placeholder]

AWS_REGION=[placeholder]
AVAILABILITY_ZONE=[placeholder]

AWS_CLI_PROFILE=[placeholder]
EC2_KEY_PAIR=[placeholder]
EC2_KEY_NAME=[placeholder]

WHITELIST_IP=[placeholder]

MINECRAFT_SEED=[placeholder]

# Leave the below as is. Bash scripts will overwrite with real values.

VOLUME_ID=[placeholder]
IP_ADDRESS=[placeholder]
INSTANCE_ID=[placeholder]
```

### Create EC2 Key Pair

#### Load environment variables

```bash
export $(grep -v '^#' .env | xargs)
```

#### Create KeyPair

```bash
aws ec2 create-key-pair --key-name $EC2_KEY_NAME --query 'KeyMaterial' --output text > $EC2_KEY_NAME.pem

chmod 400 $EC2_KEY_NAME.pem
```

### Create Stacks

#### Create EBS Stack

```bash
aws cloudformation create-stack --stack-name $EBS_STACK_NAME --template-body file://ebs_stack.yaml --parameters \
  ParameterKey=VolumeSize,ParameterValue=$VOLUME_SIZE \
  ParameterKey=AvailabilityZone,ParameterValue=$AVAILABILITY_ZONE \
  --profile $AWS_CLI_PROFILE --region $AWS_REGION > /dev/null

aws cloudformation wait stack-create-complete --stack-name $EBS_STACK_NAME --profile $AWS_CLI_PROFILE
```

#### Update VOLUME_ID in .env

```bash
VOLUME_ID=$(aws cloudformation describe-stacks --stack-name $EBS_STACK_NAME --profile $AWS_CLI_PROFILE --query 'Stacks[0].Outputs[?OutputKey==`EBSVolumeId`].OutputValue' --output text)
sed -i "" "s/VOLUME_ID=.*/VOLUME_ID=$VOLUME_ID/" .env
export $(grep -v '^#' .env | xargs)
```

#### Create EC2 Stack

```bash
aws cloudformation create-stack --stack-name $EC2_STACK_NAME --template-body file://ec2_stack.yaml --parameters \
  ParameterKey=EBSVolumeId,ParameterValue=$VOLUME_ID \
  ParameterKey=EC2KeyName,ParameterValue=$EC2_KEY_NAME \
  ParameterKey=WhitelistIP,ParameterValue=$WHITELIST_IP \
  ParameterKey=AvailabilityZone,ParameterValue=$AVAILABILITY_ZONE \
  ParameterKey=MinecraftSeed,ParameterValue=$MINECRAFT_SEED \
  ParameterKey=InstanceSize,ParameterValue=$INSTANCE_SIZE \
  --profile $AWS_CLI_PROFILE --capabilities CAPABILITY_NAMED_IAM --region $AWS_REGION > /dev/null

aws cloudformation wait stack-create-complete --stack-name $EC2_STACK_NAME --profile $AWS_CLI_PROFILE
```

#### Update IP Address in .env

```bash
IP_ADDRESS=$(aws ec2 describe-instances --filter "Name=tag:aws:cloudformation:stack-name,Values=$EC2_STACK_NAME" --query "Reservations[*].Instances[*].PublicIpAddress" --output text)
sed -i "" "s/IP_ADDRESS=.*/IP_ADDRESS=$IP_ADDRESS/" .env
export $(grep -v '^#' .env | xargs)
```

### SSH in to server

```bash
ssh -i ./$EC2_KEY_NAME.pem ec2-user@$IP_ADDRESS
```

You can check on the status of your world by running:

```bash
docker logs -f mc
```

Feel free at this point to modify the server settings. Any changes to the minecraft image configurations would require a restart of the docker container. At this point, it would be wise to consult the `itzg/minecraft-server` documenation. 

### Connect to the server

Your IP Address will be available in the `.env` file. Copy that in to your Minecraft client to load the world in multiplayer world and give it a meaningful name. Happy gaming.

### Stop and Start the Container

Stop the server when not in use to avoid un-nessecary charges, then start it again when you want to resume gaming.

#### Update Instance ID in .env

```bash
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:aws:cloudformation:stack-name,Values=$EC2_STACK_NAME" "Name=instance-state-name,Values=running" --query 'Reservations[*].Instances[*].[InstanceId]' --output text)
sed -i "" "s/INSTANCE_ID=.*/INSTANCE_ID=$INSTANCE_ID/" .env
export $(grep -v '^#' .env | xargs)
```

#### Stop the server

```bash
aws ec2 stop-instances --instance-ids $INSTANCE_ID  > /dev/null
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID
```

#### Start the server

```bash
aws ec2 start-instances --instance-ids $INSTANCE_ID  > /dev/null
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
```

### Cleanup

If you are done with the resources, simply delete the stacks. Note, this will delete the world pernamently. Consider backing up the world before doing this. 

- To-Do: Provide instructions for storing a backup in S3 and add support to restore from a backup.

#### Delete EC2 Stack

This must be done prior to deleting the EBS Stack.

```bash
aws cloudformation delete-stack --stack-name $EC2_STACK_NAME --profile $AWS_CLI_PROFILE > /dev/null
aws cloudformation wait stack-delete-complete --stack-name $EC2_STACK_NAME --profile $AWS_CLI_PROFILE
```

#### Delete EBS Stack

```bash
aws cloudformation delete-stack --stack-name $EBS_STACK_NAME --profile $AWS_CLI_PROFILE > /dev/null
aws cloudformation wait stack-delete-complete --stack-name $EBS_STACK_NAME --profile $AWS_CLI_PROFILE
```
