# Cognito Identity Pool Authentication
Use this action to perform authentication with an Amazon Cognito Identity Pool using the GitHub Actions OIDC access token. More details about this use-case can be read [here](https://catnekaise.github.io/github-actions-abac-aws/cognito-identity/).

This action supports both `Enhanched (Simplified) AuthFlow` and the `Basic (Classic) AuthFlow`.

### Table of Contents

* [Usage](#usage)
* [Standard Input Parameters](#standard-input-parameters)
* [Role Chain Input Parameters](#role-chain-input-parameters)
* [Output](#output)
* [Passing Claims (Session Tags)](#passing-claims-session-tags)
* [Usage - Role Chain](#usage---role-chain)
* [Environment Variables](#environment-variables)
* [Credentials Behaviour](#credentials-behaviour)
  * [Behaviour when Role Chaining](#behaviour-when-role-chaining)
* [Custom Role Arn (Enhanced AuthFlow only)](#custom-role-arn-enhanced-authflow-only)
* [Use as a Template](#use-as-a-template)
* [Use with a wrapper](#use-with-a-wrapper)
* [Contributing](#contributing)

## Usage

```yaml
on:
  workflow_dispatch:
jobs:
  job1:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: "Authenticate using Enhanced AuthFlow"
        uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "enhanced"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-account-id: "111111111111"
          aws-region: "eu-west-1"
          audience: "cognito-identity.amazonaws.com" # Default
          set-in-environment: true

      - name: "Authenticate using Basic AuthFlow"
        uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "basic"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          role-arn: "arn:aws:iam::111111111111:role/GhaCognito"
          aws-account-id: "111111111111"
          aws-region: "eu-west-1"
          audience: "cognito-identity.amazonaws.com" # Default
          set-in-environment: true
          
      - name: "STS Get Caller Identity"
        run: |
          aws sts get-caller-identity
```

## Standard Input Parameters

- `cognito-identity-pool-id` and `auth-flow` are required.
- `aws-account-id` and `aws-region` are required, but values can optionally be derived from environment variables, if this behaviour is wanted.
- If providing `role-arn` and `auth-flow` is `enhanced`, then `aws-account-id` can be extracted from the ARN.
- Input to action via parameters will supersede environment variables for `aws-account-id` and `aws-region`.
  - **Env Var 1** supersedes **Env Var 2**.

| Input Parameter Name     | Default                        | Example                             | Env Var 1                 | Env Var 2          |
|--------------------------|--------------------------------|-------------------------------------|---------------------------|--------------------|
| cognito-identity-pool-id | -                              | eu-west-1:11111111-example          | -                         | -                  |
| auth-flow                | -                              | basic or enhanced                   | -                         | -                  |
| aws-account-id           | -                              | 1111111111111                       | CK_COGNITO_AWS_ACCOUNT_ID | AWS_ACCOUNT_ID     |
| aws-region               | -                              | eu-west-1                           | AWS_REGION                | AWS_DEFAULT_REGION |
| audience                 | cognito-identity.amazonaws.com | cognito-identity.amazonaws.com      | -                         | -                  |
| role-arn                 | -                              | arn:aws:iam::111111111111:role/role | -                         | -                  |
| set-as-profile           | -                              | cache                               | -                         | -                  |
| set-in-environment       | -                              | true                                | -                         | -                  |


## Basic AuthFlow Parameters
When using `auth-flow` `basic`, the following parameters are also available.

| Input Parameter Name  | Default    |
|-----------------------|------------|
| role-session-name     | GhaActions |
| role-duration-seconds | 3600       |

## Role Chain Input Parameters
Role chaining can optionally be performed after authenticating with Cognito Identity.

- It's not possible to set both `set-in-environment` and `chain-set-in-environment` to `true`. Doing so will fail validation

| Parameter Name              | Default    | Example                                                    |
|-----------------------------|------------|------------------------------------------------------------|
| chain-role-arn              | -          | arn:aws:iam::111111111111:role/github-actions/WorkloadRole |
| chain-pass-claims           | -          | repository,actor,sha,ref,run_id                            |
| chain-set-as-profile        | -          | workload                                                   |
| chain-set-in-environment    | -          | true                                                       |
| chain-role-session-name     | GhaActions | GhaActions                                                 |
| chain-role-duration-seconds | 3600       | 3600                                                       |

## Output
If performing a role-chain, those credentials are used as output.

| Parameter             |
|-----------------------|
| aws_access_key_id     |
| aws_secret_access_key |
| aws_session_token     |
| aws_region            |

## Passing Claims (Session Tags)

It's possible to pass one or more claims from the initial role session to the next session when role chaining. This is done by specifying one or more GitHub Actions claims in `chain-pass-claims`.

> [!WARNING]
> Make sure the role being assumed allows `sts:TagSession` and that any claims being passed are validated by an `sts:AssumeRole` permissions policy. **If this is not done, it might be possible for an attacker to escalate privileges** if they manually make the same calls this action makes, but substitute their own values for the tags when chaining roles.

For example, if `chain-pass-claims` is `job_workflow_ref,repository`, then the following condition could be used in the IAM policy on the initial role:

```json
{
  "Sid": "AllowAssumptionIfTagsMatch",
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::*:role/pattern-matching-the-roles-to-allow-*",
  "Condition": {
    "Null": {
      "aws:TagKeys": "false"
    },
    "ForAllValues:StringEquals": {
      "aws:TagKeys": ["job_workflow_ref", "repository"]
    },
    "StringEquals": {
      "sts:RoleSessionName": "GitHubActions",
      "aws:PrincipalTag/Repository": "${aws:RequestTag/repository}",
      "aws:PrincipalTag/job_workflow_ref": "${aws:RequestTag/job_workflow_ref}"
    }
  }
}
```

This verifies that all the expected tags are set, and that the values match the ones we got from the GitHub token. The principal at this point is the initial role session, with `PrincipalTag` tags set by Cognito from the GitHub Actions claims. The `aws:RequestTag` tags are the tags passed by the caller of `AssumeRole` - i.e. this action or an attacker.

> [!IMPORTANT]
> The claims job_workflow_ref and environment is not available via an environment variable. Because of this, this action will parse the GitHub Actions access token already available and used in this action to find the value of these claims.
>
> This parsing will only occur if performing role-chaining and if passing the claim job_workflow_ref or environment.
> 
> Find step with id "missing_claims" in action.yml to view details.

The following claims can be passed by this action: `actor`, `actor_id`, `base_ref`, `event_name`, `head_ref`, `job_workflow_ref`, `ref`, `ref_type`, `repository`, `repository_id`, `repository_name`, `repository_owner`, `repository_owner_id`, `run_attempt`, `run_id`, `run_number`, `runner_environment`, `sha`, and `environment`.

`repository_name` is a special case. GitHub doesn't provide this as a claim, but it's derived from `repository` claim (which is of the form `repository_owner/repository_name`) as it is often used in IAM policies. As above, you **must** validate this claim. You can use a condition like:

```json
"aws:PrincipalTag/Repository": "${aws:PrincipalTag/repository_owner}/${aws:RequestTag/repository_name}"
```

to do this. Note that this requires you to also map `repository_owner` in your Cognito Identity Pool.

| Input                        | Comment                                                                |
|------------------------------|------------------------------------------------------------------------|
| repository                   | Becomes --tags Key=repository,Value=${{ github.repository }}           |
| repository,actor             | Appends Key,Value for actor claim                                      |
| repository , actor, run_id   | Whitespaces are trimmed                                                |
| repository=foo               | Explicitly set tag name. --tags Key=foo,Value=${{ github.repository }} |
| repository=foo,actor, run_id | Only change tag name for one claim                                     |


## Usage - Role Chain

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        description: Environment
jobs:
  job1:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "basic"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1"
          aws-account-id: "111111111111"
          role-arn: "arn:aws:iam::111111111111:role/GhaCognito"
          set-as-profile: "pool"
          chain-role-arn: "arn:aws:iam::222222222222:role/github-actions/WorkloadDeploymentDev"
          chain-pass-claims: "actor,ref,sha,repository,run_attempt,run_id,environment"
          chain-set-in-environment: true
      - run: |
          aws sts get-caller-identity
          aws sts get-caller-identity --profile pool
```

## Environment Variables

```yaml
on:
  workflow_dispatch:
env:
  CK_COGNITO_AWS_ACCOUNT_ID: "${{ vars.CK_COGNITO_AWS_ACCOUNT_ID }}"
  AWS_DEFAULT_REGION: "us-east-1"
  AWS_REGION: "us-west-1" # Overrides AWS_DEFAULT_REGION
jobs:
  job1:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: "Authenticate using Enhanced AuthFlow"
        uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "enhanced"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1" # Overrides AWS_REGION env var

      - name: "STS Get Caller Identity"
        run: |
          aws sts get-caller-identity
```

## Credentials Behaviour

- Credentials are always available as output parameters from the action.
  - The exception is when `auth-flow` is basic and no `role-arn` has been provided. Then only the `cognito_oidc_access_token` is available.
- Set input option `set-in-environment` to `true` and standard AWS environment variables will be set.
- Provide a profile name in input option `set-as-profile` to set credentials as a profile.
- Providing `set-in-environment` as true, and `set-as-profile` with a value will both set environment variables and a profile.
- When neither `set-in-environment` nor `set-as-profile` is provided, credentials are only available as outputs.

### Behaviour when Role Chaining
This is in addition to what is listed above.

- Action output will contain role-chained credentials
- Set input option `chain-set-in-environment` to `true` and standard AWS environment variables will be set.
  - It's not possible to set both `set-in-environment` and `chain-set-in-environment` to true. Doing so will fail validation before performing any authentication.
- Provide a profile name in input option `chain-set-as-profile` to set credentials as a profile.
- Providing `chain-set-in-environment` as `true`, and `chain-set-as-profile` with a value will both set environment variables and a profile.
- When neither `chain-set-in-environment` nor `chain-set-as-profile` is provided, credentials are only available as outputs.

```yaml
on:
  workflow_dispatch:

jobs:
  job1:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - id: aws-credentials
        name: "Credentials as output only"
        uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "enhanced"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1"
          aws-account-id: "111111111111"
        
      - name: "Credentials in environment"
        uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "enhanced"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1"
          aws-account-id: "111111111111"
          set-in-environment: true
          
      - name: "Credentials in profile: tf-state"
        uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "enhanced"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1"
          aws-account-id: "111111111111"
          set-as-profile: "tf-state"
          
      - name: STS Get Caller Identity
        run: |
          aws sts get-caller-identity
          aws sts get-caller-identity --profile tf-state

      - name: "STS Get Caller Identity"
        env:
          AWS_ACCESS_KEY_ID: "${{ steps.aws-credentials.outputs.aws_access_key_id }}"
          AWS_SECRET_ACCESS_KEY: "${{ steps.aws-credentials.outputs.aws_secret_access_key }}"
          AWS_SESSION_TOKEN: "${{ steps.aws-credentials.outputs.aws_session_token }}"
        run: |
          aws sts get-caller-identity
          
      - id: oidc_token
        name: "Get Cognito OIDC Access Token"
        uses: catnekaise/cognito-idpool-auth@main
        with:
          auth-flow: "basic"
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1"
          aws-account-id: "111111111111"

      - name: "Assume Role"
        env:
          OIDC_TOKEN: "${{ steps.oidc_token.outputs.cognito_identity_oidc_access_token }}"
          AWS_DEFAULT_REGION: "eu-west-1"
        run: |
          awsCredentials=$(aws sts assume-role-with-web-identity \
          --role-session-name "MySessionName" \
          --role-arn "arn:aws:iam::111111111111:role/GhaCognito" \
          --duration-seconds 3600 \
          --web-identity-token "$OIDC_TOKEN")

          awsAccessKeyId=$(echo "$awsCredentials" | jq -r ".Credentials.AccessKeyId")
          awsSecretAccessKey=$(echo "$awsCredentials" | jq -r ".Credentials.SecretAccessKey")
          awsSessionToken=$(echo "$awsCredentials" | jq -r ".Credentials.SessionToken")

          echo "::add-mask::$awsAccessKeyId"
          echo "::add-mask::$awsSecretAccessKey"
          echo "::add-mask::$awsSessionToken"
```

## Custom Role Arn (Enhanced AuthFlow only)
A custom role arn can be provided during authentication. Read more at https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html#using-rules-to-assign-roles-to-users.

Input `role-arn` is the custom role arn when `auth-flow` is also set to `enhanced`.

```yaml
on:
  workflow_dispatch:
jobs:
  job1:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: "Authenticate"
        uses: catnekaise/cognito-idpool-auth@main
        with:
         cognito-identity-pool-id: "eu-west-1:11111111-example"
         auth-flow: "enhanced"
         role-arn: "arn:aws:iam::111111111111:role/GhaCognitoCustom"
         set-in-environment: true
         aws-region: "eu-west-1"
         aws-account-id: "111111111111"
         
      - name: STS Get Caller Identity
        run: |
          aws sts get-caller-identity
```

## Use as a Template
Copy and paste the existing action.yml in this repository to your own repository and customize it as you see fit.

## Use with a wrapper
Use the "wrapper-template" below and pin using of this action to a specific commit.

Optionally set one or more defaults in inputs to reduce amount of repetitive input.

```yaml
name: Wrapped Cognito Identity Auth
inputs:
  aws-account-id:
    required: false
    # Optionally set defaults in inputs
    default: "111111111111"
    description: "AWS Account ID"
  # Copy remaining inputs from ./action.yml
outputs:
  aws_access_key_id:
    description: "AWS Access Key Id"
    value: ${{ steps.auth.outputs.aws_access_key_id }}
  aws_secret_access_key:
    description: "AWS Secret Access Key"
    value: ${{ steps.auth.outputs.aws_secret_access_key }}
  aws_session_token:
    description: "AWS Session Name"
    value: ${{ steps.auth.outputs.aws_session_token }}
  aws_region:
    description: "AWS Region"
    value: ${{ steps.auth.outputs.aws_region }}
  cognito_identity_oidc_access_token:
    description: "Cognito Identity OIDC Access Token"
    value: ${{ steps.auth.outputs.cognito_identity_oidc_access_token }}
runs:
  using: composite
  steps:
    - id: auth
      uses: catnekaise/cognito-idpool-auth@SHA_HERE
      with: 
        cognito-identity-pool-id: "${{ inputs.cognito-identity-pool-id }}"
        auth-flow: "${{ inputs.auth-flow }}"
        aws-region: "${{ inputs-aws-region }}"
        audience: "${{ inputs.audience }}"
        aws-account-id: "${{ inputs.aws-account-id }}"        
        role-arn: "${{ inputs.role-arn }}"
        role-duration-seconds: "${{ inputs.role-duration-seconds }}"
        role-session-name: "${{ inputs.role-session-name }}"
        set-as-profile: "${{ inputs.set-as-profile }}"
        set-in-environment: "${{ inputs.set-in-environment }}"
        chain-role-arn: "${{ inputs.chain-role-arn }}"
        chain-role-duration-seconds: "${{ inputs.chain-role-duration-seconds }}"
        chain-role-session-name: "${{ inputs.chain-role-session-name }}"
        chain-pass-claims: "${{ inputs.chain-pass-claims }}"
        chain-set-as-profile: "${{ inputs.chain-set-as-profile }}"
        chain-set-in-environment: "${{ inputs.chain-set-in-environment }}"
```

## Contributing
Contributions are welcome. This action can use more testing and reviews of the large amount of shell scripting.
