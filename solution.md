To find the flag, I’ve taken the following steps:

•Step 1: Environment Setup
  I started by cloning and deploying the lab environment using Terraform. This created the required IAM users, roles, and Lambda-related resources. 
  -git clone https://github.com/humbercloudsecurity/ccgc5501-cloud-security-challenge-lambda_privesc
  -cd ccgc5501-cloud-security-challenge-lambda_privesc
  -terraform init
  -terraform plan
  -terraform apply
  After deployment, I retrieved the access keys for the chris user and configured a named AWS CLI profile.

•Step 2: Verify Initial Identity
  Using the configured profile, I verified that I was operating as the chris user and reviewed the policies attached to this account.
  -aws sts get-caller-identity --profile chris
  -aws iam list-attached-user-policies --user-name chris-lambda_privesc --profile chris
  This confirmed that chris did not have administrator privileges.

•Step 3: Identify a Privileged Role
  Next, I listed the available IAM roles and identified a role named lambdaManager, which had permissions related to AWS Lambda.
  -aws iam list-roles --profile chris
  -aws iam get-role --role-name lambdaManager --profile chris

•Step 4: Assume the lambdaManager Role
  I assumed the lambdaManager role using AWS STS and exported the temporary credentials into my environment.
  -aws sts assume-role --role-arn arn:aws:iam::306989527055:role/lambdaManager-role-lambda_privesc
  After setting the credentials, I verified that my identity had changed to the assumed role.
  -aws sts get-caller-identity

•Step 5: Create a Malicious Lambda Function
  While operating as lambdaManager, I created a Lambda function designed to attach the AdministratorAccess policy to the chris-lambda_privesc user.
  The Lambda function used the IAM API to attach the policy programmatically.
  
•Step 6: Invoke the Lambda Function
  I deployed and invoked the Lambda function, which successfully attached the administrator policy to the chris user.
  -aws lambda invoke --function-name PrivilegeEscalation response.json
  The response confirmed successful execution.

•Step 7: Verify Privilege Escalation
  After clearing the assumed role credentials, I checked the policies attached to the chris user again.
  -aws iam list-attached-user-policies --user-name chris --profile chris
  The AdministratorAccess policy was now attached. I confirmed full admin privileges by listing all IAM users.
  -aws iam list-users --profile chris

REFLECTION

Question 1: What was your approach?

~ I started by setting up the lab environment and verifying my initial permissions. I then looked for roles that could be abused and found a Lambda-related role with elevated privileges. By assuming that role, I was able to create and execute a Lambda function that escalated my access.

Question 2: What was the biggest challenge?

~ The biggest challenge was troubleshooting why the Lambda function was failing even though the permissions appeared correct. The issue turned out to be a username mismatch, where the Lambda code referenced chris instead of the actual IAM user chris-lambda_privesc.

Question 3: How did you overcome the challenge?

~ I reviewed the IAM commands and outputs carefully and compared them with the Lambda code. By identifying the correct IAM username used in the lab environment, I updated the Lambda function accordingly. This fix allowed the privilege escalation to succeed.

Question 4: What led to the breakthrough?

~ The breakthrough happened when the Lambda function successfully attached the AdministratorAccess policy to the chris user, confirming that the privilege escalation worked.

Question 5: On the blue side, how can this learning be used to properly defend important assets?

~ This challenge highlights the risks of overly permissive roles and Lambda access. To defend important assets:

  -Apply least privilege to IAM roles
  
  -Restrict who can create or invoke Lambda functions
  
  -Monitor role assumptions using CloudTrail
  
  -Regularly audit IAM policies and role trust relationships

