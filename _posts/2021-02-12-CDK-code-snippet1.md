---
layout: post
title: "CDK code snippet-1"
date: 2021-02-12
excerpt: "I used the Terraform for IaC but the HCL is very difficult to learn and maintain because it uses the JSON-like format and we need to lean the new language"
tags: [AWS, CDK, Python]
comments: false
---

# CDK code snippet

## Introduction

Recently, I'm using the [CDK](https://aws.amazon.com/cdk/?nc1=h_ls) for IaC, Infrastructure as Code. I used the Terraform for IaC but the HCL is very difficult to learn and maintain because it uses the JSON-like format and we need to lean the new language. You can get more introduce and reference of HCL from [this documents](https://www.terraform.io/docs/language/index.html).

But CDK is easy to read the code because CDK represents the infrastructure with TypeScript, Python, and C#. It means we don't need to learn very specific language and good to build the code structure like a common programming language.

However, I got the struggle with CDK too because it's a little bit early project so sometimes no reference or guide even AWS has [the official documentation site](https://docs.aws.amazon.com/cdk/api/latest/). So I made the code snippet for each usage. Moreover, I wrote the code in Python. Maybe you can find the TypeScript example easily but difficult for other languages because TypeScript is the main language for CDK.

-------
<br>

## Create user and get the access and secret

`create_aws_user()` creates the user with policy

```python
from aws_cdk import (
    aws_iam as iam,
    core
)

def create_aws_user(resource: core.Resource, 
                    user_id: str, access_key_id: str,
                    managed_policies: Optional[List[iam.IManagedPolicy]])->(iam.User, iam.CfnAccessKey):
        """ Create the user and returns the user and access key
				:param resource: core Resource
        :param user_id: ID of AWS user resource, must be unique
        :param access_key_id: ID of access key resource
        :param managed_policies: Managed policies which AWS user has
				:rtype: (am.User, iam.CfnAccessKey)
        """
        user = iam.User(resource, user_id, managed_policies=managed_policies)
        access_key = iam.CfnAccessKey(resource, access_key_id, user_name=user.user_name)
			  return user, access_key
```

You can use this core like following.

```python

from aws_cdk import (
    aws_iam as iam,
    core
)

class MyStack(core.Stack):

    def __init__(self, scope: core.Construct, id: str, name_tag: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)
        ......

        # Setup aws user for send data to firehose
        firehose_user_id = f"My-Firehose-User"
        firehose_access_key_id = f"My-access-key"
        firehose_user, firehose_access_key = \
          create_aws_user(firehose_user_id,
              firehose_access_key_id, [
                iam.ManagedPolicy.from_aws_managed_policy_name(
                  'AmazonKinesisFirehoseFullAccess'
                  ),
              ]
            )

		# Access key : firehose_access_key.ref
		# Access secret : firehose_access_key.attr_secret_access_key
```

<br>

--------

## Create S3 and prepare the CloudFront for public

S3 is a very popular service and there are so many various use cases. In my case, I wanna configure S3 for public or private. Especially, I serve the file object with my own domain, not the S3 domain through HTTPS protocol for the public. Using HTTPS and my own domain is good practice for public file serving.

I implemented the code like following.

```python
from aws_cdk import (
    aws_s3 as s3,
    aws_iam as iam,
    aws_route53 as route53,
    aws_route53_targets as targets,
    aws_cloudfront as cf,
    core
)

def setup_s3_with_domain_and_ssl_with_user(resource: core.Resource,
                                           s3_id: str,
                                           user_to_access_s3_id: str,
                                           public_read_access: bool = False,
                                           domain_for_storage: str = None,
                                           cf_distribution_id: str = None,
                                           hosted_zone_id: str = None) -> (s3.Bucket, iam.User):
    """ Set up S3 with SSL and domain.

    :param resource: CDK core resource
    :param s3_id: ID of S3 resource, must be unique
    :param user_to_access_s3_id: ID of user to access S3,  must be unique
    :param public_read_access: Allow the public read access or not. Default is False.
    :param domain_for_storage: Domain to link S3 with CDN,  must be unique. Default is None.
    :param cf_distribution_id: ID of CloudFront(CDN),  must be unique. Default is None.
    :param hosted_zone_id: ID of Route53 hosted zone, must be unique. Default is None.
    :rtype: (s3.Bucket, iam.User)
    """
    s3_bucket = None

    if public_read_access:
        s3_bucket = s3.Bucket(
            resource, s3_id,
            bucket_name=s3_id.lower(),
            access_control=s3.BucketAccessControl.PUBLIC_READ,
            public_read_access=public_read_access,
        )
    else:
        s3_bucket = s3.Bucket(
            resource, s3_id,
            bucket_name=s3_id.lower(),
            block_public_access=s3.BlockPublicAccess(
                block_public_acls=True,
                block_public_policy=True,
                ignore_public_acls=True,
                restrict_public_buckets=True,
            )
        )

    # Grant read access to everyone in your account for debugging.
    s3_bucket.add_to_resource_policy(
        iam.PolicyStatement(
            actions=['s3:GetObject'],
            resources=[s3_bucket.arn_for_objects("*")],
            principals=[iam.AccountPrincipal(
                account_id=core.Aws.ACCOUNT_ID)]
        )
    )

    # Grant write access to a specific user
    # - User for s3 access to write. See https://github.com/aws/aws-cdk/issues/1612,
    # https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-iam-accesskey.html#cfn-iam-accesskey-serial
    user_name = user_to_access_s3_id
    s3_user = iam.User(resource, user_name)
    s3_bucket.grant_write(s3_user)

    # Setup CDN and domain
    # Reference: https://github.com/rhboyd/aws-cdk-examples/blob/master/python/static-site/static_site/static_site_construct.py#L56-L82
    if public_read_access:
				# Pre-existing ACM Certificate, with ARN stored in an SSM Parameter
        certificate_arn = "GET_YOUR_ACM_CERTIFICATE_ARN"
        # To-Do: Using hard-coded certificate_arn. Need to use SSM as reference document mentioned.  Need to revise.

        alias_configuration = cf.AliasConfiguration(
            acm_cert_ref=certificate_arn,
            names=[domain_for_storage],
            ssl_method=cf.SSLMethod.SNI,
            security_policy=cf.SecurityPolicyProtocol.TLS_V1_1_2016
        )

        source_configuration = cf.SourceConfiguration(
            s3_origin_source=cf.S3OriginConfig(
                s3_bucket_source=s3_bucket
            ),
            behaviors=[cf.Behavior(is_default_behavior=True)]
        )

        distribution = cf.CloudFrontWebDistribution(
            resource,
            cf_distribution_id,
            alias_configuration=alias_configuration,
            origin_configs=[source_configuration]
        )

        # To-Do: Using the hard-coded one for Route53. Need to revise.
        hosted_zone = route53.HostedZone.from_hosted_zone_attributes(
            resource,
            hosted_zone_id,
            zone_name='YOUR_ZONE_NAME_ID_FROM_ROUTE53',
            hosted_zone_id='YOUR_ZONE_ID_FROM_ROUTE53'
        )

        route53.ARecord(resource,
                        domain_for_storage,
                        record_name=domain_for_storage,
                        zone=hosted_zone,
                        target=route53.AddressRecordTarget.from_alias(
                            targets.CloudFrontTarget(distribution))
                        )

    return s3_bucket, s3_user

```

You can use this function like the following.

```python
from aws_cdk import (
    aws_iam as iam,
    core
)

class MyStack(core.Stack):

    def __init__(self, scope: core.Construct, id: str, name_tag: str, **kwargs) -> None:
      super().__init__(scope, id, **kwargs)
      ......

    # Create the S3 for private with new user
    private_s3_id = "s3-for-private-with-new-user"
    user_id_for_this_s3 = "user-for-s3-for-private-with-new-user"

    (private_s3_bucket, user_for_private_s3) = \
      setup_s3_with_domain_and_ssl_with_user(self, private_s3_id,user_id_for_this_s3)

    # Create the S3 for the public with new user
    public_s3_id = "s3-for-public-with-new-user"
    user_id_for_public_s3 = "user-for-s3-for-public-with-new-user"
    cf_distribution_id ="s3-for-public-storage-distribution"
    hosted_zone_id = "s3-for-public-hosted-zone"
    cdn_domain_for_storage = "my.own.domain.com"

    # You can serve file object URL like 'https://my.own.domain.com/{file_object}'.
    (s3_for_images, s3_user) = \
      setup_s3_with_domain_and_ssl_with_user(
        self,public_s3_id,user_id_for_public_s3, True,
        cdn_domain_for_storage, cf_distribution_id, hosted_zone_id)

```

<br>

--------
## Create the user pool with your domain mail

Yes, it was the most difficult case because I need to handle CloudFront directly. CDK uses CloudFormation technology it means CDK convert your code to JSON file for CloudFormation.

I want to use Cognito user pool service. Then I need to send the notification via e-mail for verifying the user. The problem was Cognito sends the mail with the '[no-reply@verificationemail.com](mailto:no-reply@verificationemail.com)'. But I wanna use the company's domain. So I have to configure [SES](https://aws.amazon.com/ses/?nc1=h_ls) first. And SES should be in Virginia or Oregon region.

```python
from aws_cdk import (
    aws_cognito as cognito,
    core
)

def setup_service_userpool(resource: core.Resource, userpool_id: str, email_body: str, custom_attributes: dict, self_sign_up_enabled: bool = False) -> cognito.UserPool:
    """ Setup cognito user pool
    :param resource : CDK core resource
    :param userpool_id : Unique id of this userpoo;
    :param email_body : Email body to send
    :param custom_attributes : Attribution of user
    :param self_sign_up_enabled : Whether accept to sign up by user or not. Default is False
    :rtype: cognito.UserPool
    """

    # SES ARN : To-Do Need to prepare SES 1st.
    email_source_arn = "USER_SES_ARN"

    cog = cognito.UserPool(
        resource, userpool_id,
        account_recovery= cognito.AccountRecovery(cognito.AccountRecovery.EMAIL_AND_PHONE_WITHOUT_MFA),
        auto_verify=cognito.AutoVerifiedAttrs(
            email=True),
        custom_attributes=custom_attributes,
        email_settings=None,
        lambda_triggers=None,
        password_policy=cognito.PasswordPolicy(
            min_length=6,
            require_digits=True,
            require_lowercase=True,
            temp_password_validity=core.Duration.days(1)
        ),
        self_sign_up_enabled=self_sign_up_enabled,
        user_pool_name=userpool_id,
        sign_in_aliases=cognito.SignInAliases(
            email=True
        ),
        user_verification=cognito.UserVerificationConfig(
            email_body=email_body,
            email_style=cognito.VerificationEmailStyle.CODE,
            email_subject='[Verification] A verification code has arrived from Youha'
        )
    )

    email_conf = cognito.CfnUserPool.EmailConfigurationProperty(
        email_sending_account="DEVELOPER",
        from_="YOUR_VERIFIED_EMAIL_ADDRESS",
        source_arn=email_source_arn)
    cfn_userpool = cog.node.default_child
    cfn_userpool.email_configuration = f"{userpool_id}_email_conf"

    cog.add_client(userpool_id + "-App")
    return cog
```

Then you can use this function like the following.

```python
from aws_cdk import (
    aws_iam as iam,
    core
)

class MyStack(core.Stack):

    def __init__(self, scope: core.Construct, id: str, name_tag: str, **kwargs) -> None:
      super().__init__(scope, id, **kwargs)
      ......

    # User pool with self sign up
    user_pool_with_self_sign_up = \
      setup_service_userpool(self, 
        userpool_id = 'user_pool_without_self_sign_up',
        custom_attributes={
          "userId": cognito.StringAttribute(max_len=128, min_len=1, mutable=True),
          "userRole": cognito.StringAttribute(max_len=128, min_len=1, mutable=True)},
          self_sign_up_enabled=True
      )
```

## Not perfect, need to customize!

 I'm just sharing the code snippet because maybe I can help. ;-) NEVER COPY AND PASTE THIS CODE and understand how it works. And Good luck! 