# cognito-idpool-auth
Use this action to perform authentication with a Amazon Cognito Identity Pool using the GitHub Actions OIDC access token.

This action is when using `Enhanched (Simplified) AuthFlow`. A different action for `Basic (Classic) AuthFlow` is available [here](https://github.com/catnekaise/cognito-idpool-basic-auth).

### Use as Template
This repository is available as a template repository.

## Alpha Status
At the time writing (October 2023) this action has not been tested or reviewed by anyone other than the author, hence the alpha status. If using this action, please provide feedback.

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
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-account-id: "111111111111"
          aws-region: "eu-west-1"
          audience: "cognito-identity.amazonaws.com"
          set-in-environment: true
          
      - name: "STS Get Caller Identity"
        run: |
          aws sts get-caller-identity
```

### Input Parameters

- Only `cognito-identity-pool-id` is a required input to the action. 
- `aws-account-id` and `aws-region` are required, but values can optionally be derived from environment variables, if this behaviour is wanted.
- Input to action via parameters will supersede environment variables for `aws-account-id` and `aws-region`.
  - **Env Var 1** supersedes **Env Var 2**.

| Input Parameter Name     | Default value                  | Example                             | Env Var 1                 | Env Var 2          |
|--------------------------|--------------------------------|-------------------------------------|---------------------------|--------------------|
| cognito-identity-pool-id | -                              | eu-west-1:11111111-example          | -                         | -                  |
| aws-account-id           | -                              | 1111111111111                       | CK_COGNITO_AWS_ACCOUNT_ID | AWS_ACCOUNT_ID     |
| aws-region               | -                              | eu-west-1                           | CK_COGNITO_AWS_REGION     | AWS_DEFAULT_REGION |
| audience                 | cognito-identity.amazonaws.com | cognito-identity.amazonaws.com      | -                         | -                  |
| custom-role-arn          | -                              | arn:aws:iam::111111111111:role/role | -                         | -                  |
| set-as-profile           | -                              | cache                               | -                         | -                  |
| set-in-environment       | -                              | true                                | -                         | -                  |

### Outputs

| Parameter             |
|-----------------------|
| aws_access_key_id     |
| aws_secret_access_key |
| aws_session_token     |
| aws_region            |




### Env Vars Example

```yaml
on:
  workflow_dispatch:
env:
  CK_COGNITO_AWS_ACCOUNT_ID: "${{ vars.CK_COGNITO_AWS_ACCOUNT_ID }}"
  AWS_DEFAULT_REGION: "us-east-1"
jobs:
  job1:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: "Authenticate using Enhanced AuthFlow"
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          aws-region: "eu-west-1" # Overrides AWS_DEFAULT_REGION

      - name: "STS Get Caller Identity"
        run: |
          aws sts get-caller-identity
```

### Handling Credentials

- Credentials are always available as output parameters from the action.
- Set input option `set-in-environment` to `true` and standard AWS environment variables will be set.
- Provide a profile name in input option `set-as-profile` to set credentials as a profile.
- Providing `set-in-environment` as true, and `set-as-profile` with a value will both set environment variables and a profile.
- When neither `set-in-environment` nor `set-as-profile` is provided, credentials are only available as outputs.

```yaml
on:
  workflow_dispatch:
env:
  CK_COGNITO_AWS_ACCOUNT_ID: "${{ vars.CK_COGNITO_AWS_ACCOUNT_ID }}"
  AWS_DEFAULT_REGION: "eu-west-1"
jobs:
  job1:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - id: aws-credentials
        name: "Credentials as output only"
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
          cognito-identity-pool-id: "eu-west-1:11111111-example"
        
      - name: "Credentials in environment"
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
          cognito-identity-pool-id: "eu-west-1:11111111-example"
          set-in-environment: true
          
      - name: "Credentials in profile: tf-state"
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
          set-as-profile: "tf-state"
          cognito-identity-pool-id: "${{ vars.TF_STATE_COGNITO_IDENTITY_POOL_ID }}"
          
      - name: "Credentials in profile: cache"
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
          set-as-profile: "cache"
          cognito-identity-pool-id: "${{ vars.CACHE_COGNITO_IDENTITY_POOL_ID }}"
          
      - name: STS Get Caller Identity
        run: |
          aws sts get-caller-identity
          aws sts get-caller-identity --profile tf-state
          aws sts get-caller-identity --profile cache

      - name: "STS Get Caller Identity"
        env:
          AWS_ACCESS_KEY_ID: "${{ steps.aws-credentials.outputs.aws_access_key_id }}"
          AWS_SECRET_ACCESS_KEY: "${{ steps.aws-credentials.outputs.aws_secret_access_key }}"
          AWS_SESSION_TOKEN: "${{ steps.aws-credentials.outputs.aws_session_token }}"
        run: |
          aws sts get-caller-identity
```

### Custom Role Arn
A custom role arn can be provided during authentication. Read more at https://docs.aws.amazon.com/cognito/latest/developerguide/role-based-access-control.html#using-rules-to-assign-roles-to-users.

```yaml
on:
  workflow_dispatch:
jobs:
  job1:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: "Authenticate using ID Pool"
        uses: catnekaise/cognito-idpool-auth@alpha
        with:
         custom-role-arn: "arn:aws:iam::111111111111:role/role-name"
         cognito-identity-pool-id: "${{ vars.COGNITO_IDENTITY_POOL_ID }}"
         set-in-environment: true
         aws-region: "eu-west-1"
         aws-account-id: "111111111111"
         
      - name: STS Get Caller Identity
        run: |
          aws sts get-caller-identity
```
