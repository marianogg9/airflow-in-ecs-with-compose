# Airflow in AWS ECS with docker compose

This repository contains all required code to deploy Airflow in ECS using `docker compose` [integration](https://docs.docker.com/cloud/ecs-integration/).

Please check out this [Hashnode article I wrote](https://blog.mariano.cloud/airflow-in-ecs-with-redis-part-3-docker-compose) for a walkthrough and an example DAG run.

I have included an example DAG in `example-dag/process-employees.py` taken from [Airflow's example pipeline documentation](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/pipeline.html).

## Prerequisites
- Install [latest](https://www.docker.com/products/docker-desktop/) Docker Desktop.

- AWS account.
  - VPC + subnet(s).
  - IAM credentials with the following permissions:
    - [Base](https://docs.docker.com/cloud/ecs-integration/#requirements).
    - Additionally:
      - `ec2:DescribeVpcAttribute`.
      - `elasticfilesystem:Describe*`.
      - `elasitcfilesystem:Create*`.
      - `elasticfilesystem:Delete*`.
      - `logs:TagResource`.
      - `iam:PutRolePolicy`.
      - `iam:DeleteRolePolicy`.

- Create a security group (and rules) to be used in this deployment.
  ```bash
  aws ec2 create-security-group \
      --group-name Airflow --description "Airflow traffic" \
      --vpc-id "<your-vpc-id>"
  ```
  It creates an egress rule to `0.0.0.0/0` by default and outputs the following (take note of the ID for next steps):

  ```bash
  {
    "GroupId": "<airflow_SG_above_created_id>"
  }
  ```

  Create internal traffic (ingress) rules:
  - Self traffic (between services):
  ```bash
  aws ec2 authorize-security-group-ingress \
      --group-id "<airflow_SG_above_created_id>" \
      --protocol all \
      --source-group "<airflow_SG_above_created_id>"
  ```

  - Inter VPC (NLB health checks):
  ```bash
  aws ec2 authorize-security-group-ingress \
      --group-id "<airflow_SG_above_created_id>" \
      --ip-permissions IpProtocol=-1,FromPort=-1,ToPort=-1,IpRanges="[{CidrIp=<your_vpc_cidr>,Description='Allow VPC internal traffic'}]"
  ```
  
  Create external traffic (ingress) rule (so you can access both Webserver and Flower UI):
  ```bash
  aws ec2 authorize-security-group-ingress \
      --group-id <airflow_SG_above_created_id> \
      --ip-permissions IpProtocol=tcp,FromPort=5555,ToPort=5555,IpRanges="[{CidrIp=<your_public_CIDR>,Description='Allow Flower access'}]" IpProtocol=tcp,FromPort=8080,ToPort=8080,IpRanges="[{CidrIp=<your_public_CIDR>,Description='Allow Webserver access'}]"
  ```

**NOTE:** There is **currently** no way (natively) of avoiding CloudFormation creating a `0.0.0.0/0` rule in the SG for `5555` and `8080`. See [issue](https://github.com/docker/compose-cli/issues/1783).
If you need to narrow down access, you will have to delete these two additional rules when the SG rules are created in a later step (while `docker compose up` creates `airflow-webserver` and `airflow-flower` services).

## Deploy

- Clone this repo:
  ```bash
  git clone repo local_dir
  cd local_dir
  ```

- Set required variables in `docker-compose.yaml`:
  ```yaml
  x-aws-vpc: "your VPC id"
  networks:
    back_tier:
      external: true
      name: "<airflow_SG_above_created_id>"
  ```

- (optional) If you want to use a custom password for the Webserver admin user (default user `airflow`):

  This password will be created as an AWS Secrets Manager secret and its ARN will be passed as an environment variable with the following format:
  ```yaml
  secrets:
    name: _AIRFLOW_WWW_USER_PASSWORD
    valueFrom: <secrets_manager_secret_arn>
  ```
  - Add required environment variables to `airflow-webserver` service:
    ```yaml
    environment:
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: 'airflow'
    ```
  - Add a custom password in a local file:
    ```bash
    echo 'your_custom_password' > ui_admin_password
    ```
  - Add a `secrets` definition block in `docker-compose.yaml`:
    ```yaml
    secrets:
        ui_admin_password: 
            name: _AIRFLOW_WWW_USER_PASSWORD
            file: ./ui_admin_password
    ```
  - Add a `secrets` attribute `airflow-webserver` compose service:
    ```yaml
    secrets:
      - _AIRFLOW_WWW_USER_PASSWORD
    ```
  - Add the following required AWS Secrets Manager permissions to the IAM credentials you set `docker context` to use:
    - `secretsmanager:CreateSecret`
    - `secretsmanager:DeleteSecret`
    - `secretsmanager:GetSecretValue`
    - `secretsmanager:DescribeSecret`
    - `secretsmanager:TagResource`
    - `secretsmanager:UpdateSecret`
    
    Narrow down the above permissions to the secret ARN: `arn:aws:secretsmanager:<your_aws_region>:<your_aws_account_id>:secret:AIRFLOWWWWUSERPASSWORD*`

- Create a new `docker` context:
  ```bash
  docker context create ecs new-context-name
  ```
  - Select your preferred method of obtaining credentials (env vars, profile or secret:token):

- Use the newly crated `docker` context:
  ```bash
  docker context use new-context-name
  ```

- (optional) Review CloudFormation template before deploying:
  ```bash
  docker compose convert
  ```

- Deploy:
  ```bash
  docker compose up
  ```

  This command will start showing updates on screen. You can also follow up the resources creation on AWS CloudFormation console.

## Web access
- Get NLB CNAME:
  ```bash
  aws elbv2 describe-load-balancers | grep DNSName | awk '{print$2}' | sed -e 's|,||g'
  ```
  (if you have `jq` installed: `aws elbv2 describe-load-balancers | jq .LoadBalancers[].DNSName`)

  Or get the DNSName from AWS console directly.

- To access Webserver UI: `http://<NLB_DNSName>:8080`

  Authenticate with default `airflow:airflow` or the custom password you set using secret `_AIRFLOW_WWW_USER_PASSWORD`.

- To access Flower UI: `http://<NLB_DNSName>:5555`

## Cleanup

- Once you are done, you can delete all created resources running:
    ```bash
    docker compose down
    ```

    Same as with `up`, CloudFormation stack updates will be shown on screen.
    ```
    [+] Running 5/9
    ⠋ airflow-ecs-compose                 DeleteInProgress User Initiated                               88.1s
    ```

- Delete the custom Security Group.
  - First delete the ingress rules:
    ```bash
    aws ec2 revoke-security-group-ingress \
        --group-id "<airflow_SG_above_created_id>" \
        --security-group-rule-ids "<self_internal_SG_rule_above_created_id>"
    ```

    ```bash
    aws ec2 revoke-security-group-ingress \
        --group-id "<airflow_SG_above_created_id>" \
        --ip-permissions IpProtocol=-1,FromPort=-1,ToPort=-1,IpRanges="[{CidrIp=<your_vpc_cidr>,Description='Allow VPC internal traffic'}]"
    ```

    ```bash
    aws ec2 revoke-security-group-ingress \
        --group-id <airflow_SG_above_created_id> \
        --ip-permissions IpProtocol=tcp,FromPort=5555,ToPort=5555,IpRanges="[{CidrIp=<your_public_CIDR>,Description='Allow Flower access'}]" IpProtocol=tcp,FromPort=8080,ToPort=8080,IpRanges="[{CidrIp=<your_public_CIDR>,Description='Allow Webserver access'}]"
    ```
  - Then the SG itself:
    ```bash
    aws ec2 delete-security-group \
        --group-id "<airflow_SG_above_created_id>"
    ```


**Important: delete EFS volumes mannually!!**
`docker compose` integration creates those volumes with `retain` policy so that whenever a new deployment occurs, it can reuse them. From `docker compose down` outputs:

```
AirflowFilesystem                                        DeleteSkipped
PostgresdbvolumeFilesystem                               DeleteSkipped
```

So, don't forget to delete those EFS volumes.
