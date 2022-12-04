# Scenario #1: Flexible Trust Relationships

## GitHub Workflow
Here's an excerpt from Scenario #1's workflow which simple provides the relevant permissions to the job and then proceeds to configure the AWS credentials using the OpenID Connect provider. 

```
...
    # These permissions are needed to interact with GitHub's OIDC Token endpoint.
    permissions:
      id-token: write
      contents: read

    - name: Configure AWS credentials for the dummy account (Sandbox)
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: arn:aws:iam::975216159326:role/S3Admin
        aws-region: us-east-1
...
```

## AWS OIDC Provider and Trust Relationship
As part of the OIDC workflow, AWS uses the `subject` and `audience` (along with other claims part of the JWT token sent by GitHub) for it to validate the authenticity of the request. Here's the configured trust relationship of the role which is assumed as part of the Action/Workflow presented above:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Principal": {
                "Federated": "arn:aws:iam::648765633154:oidc-provider/token.actions.githubusercontent.com"
            },
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": [
                        "sts.amazonaws.com"
                    ]
                }
            }
        }
    ]
}
```

# Solution
Currently, the Trust Relationship doesn't have any `trust conditions` suggesting 


