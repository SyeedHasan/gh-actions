# Scenario # 1: Flexible Trust Relationships

## Scenario #1: Flexible Trust Relationships

### GitHub Workflow

Here's an excerpt from **Scenario #1's workflow** which simply provides the relevant permissions to the job and then proceeds to configure the AWS credentials using the OpenID Connect provider:

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

### AWS OIDC Provider and Trust Relationship

As part of the OIDC workflow, AWS validates the request by comparing the claims it has receives versus the trust relationship managed by the account owner. Here's the configured trust relationship of the role which is assumed as part of the Action presented above:

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

## Solution

In the implementation of the `trust relationship` shared above, there's no conditional check or validation to understand if the sender of the JWT token from the OIDC provider is the same subject which was authorised by the account owner to assume the role/dynamically retrieve short-term credentials against the federated user (the job in this case).

As such, anyone can now assume this role from wherever they want and perform the actions available to the role (permitted by the policies assigned to the role).&#x20;

The solution is to **harden the trust relationship** between the cloud provider (AWS) and the IdP (GitHub)**.** That's possibly by adding more conditionals for the provider to validate the request against. If all checks match, the request is successful and the credentials are assumed by the GitHub worker/job.&#x20;

For instance, we can add the `sub` condition to the trust policy to ensure we're only allowing certain repositories, accounts, or environments to be able to authenticate (the wildcard allows any branch or environment from the configured account and repository to be able to authenticate):&#x20;

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::648765633154:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:sub": "repo:SyeedHasan/gh-actions:*"
                }
            }
        }
    ]
}
```

A slightly different way (_yet not entirely effective_) to mitigate the risk of this is to ensure the `audience` sent as part of the claims isn't default (as suggested in the guidebook of the [`configure-aws-credentials`](https://github.com/aws-actions/configure-aws-credentials) action.  A different audience field ensures that the request is only authenticated if the correct audience is delivered using the workflow.&#x20;

GitHub's [documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#customizing-the-token-claims) on hardening the OpenID connector's and workflows further explains the use-cases of customizing the claims and the subject (`sub`) of the requests as well.&#x20;

