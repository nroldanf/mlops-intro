# Guide

## Configuring GH actions with OIDC GitHub <> AWS with Assumerole

To create a GitHub OpenID Connect (OIDC) provider in AWS along with a GitHub actions workflow example on how to authenticate to AWS and access resources using the aws-actions/configure-aws-credentials follow the next steps.

1. Open the IAM console at https://console.aws.amazon.com/iam/
2. In the navigation pane, choose `Identity providers`, and then choose `Add provider`.
3. For Configure provider, choose OpenID Connect.
    For Provider URL: https://token.actions.githubusercontent.com
    For Audience: sts.amazonaws.com
4. Add Provider.

The next step is to create the appropriate IAM Role with least privileged policies to the desired AWS service (eg. S3,ECR)

1. Open the IAM console at https://console.aws.amazon.com/iam/
2. In the navigation pane, choose Roles, and then choose Create Role.
3. For trusted entity select `Web Identity` and for Identity provider select `token.actions.githubactions.com` as for the Audience select `sts.amazonaws.com`.
4. Assign the policies you want this role to be equpped with, and on the next screen fill the trusted entities with this example below where you need to edit the appropriate settings for your use case.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::ACCOUNTID:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:LokaHQ/reponame:*"
                }
            }
        }
    ]
}
```
For multi repo permissions you can add list of repos - ex. 

```json

"StringLike": {
                    "token.actions.githubusercontent.com:sub": [
                         "repo:LokaHQ/reponame1:*",
                         "repo:LokaHQ/reponame2:*"
                         "repo:LokaHQ/reponame3:*" ]
                }
                
```

## Adding permissions settings

The job or workflow run requires a ``permissions`` setting with ``id-token: write``. You won't be able to request the OIDC JWT ID token if the ``permissions`` setting for ``id-token`` is set to ``read`` or ``none``.


If you only need to fetch an OIDC token for a single job, then this permission can be set within that job. For example:

```yaml
permissions:
  id-token: write
```
Also, for your JWT Token and for OIDC to be successful when running GH actions in the workflow file you must define the login.

```yaml

- name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-region: us-east-1
        role-to-assume: arn:aws:iam::ACCOUNTID:role/YourRole
        role-session-name: YourSession
```

## Boilerplate Example 

```yaml
on:
  push:
    branches:
    - main
    
permissions:
      id-token: write
      contents: read 

jobs:
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest

    steps:
    - name: Check out
      uses: actions/checkout@v2

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-region: us-east-1
        role-to-assume: arn:aws:iam::ACCOUNTID:role/YourRole
        role-session-name: YourSession

    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag, and push image to Amazon ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: Your Repository
        IMAGE_TAG: latest
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

RTFM:
* https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
* https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions
* https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html
* https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html