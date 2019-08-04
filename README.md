# Introduction to modern authentication patterns for application developers

By Jason Umiker (jumiker@amazon.com)

## Background and History (OAUTH v2)

Almost since the beginning of the Web there has been a need to secure websites to ensure not everyone on the Internet has access. The way that this is done is that a user `authenticates` with a username and password. The server then `authorizes` them to perform certain actions based on their permissions. This distinction between authentication and authorization is important - one is verifying we are who we say we are and the other is a decision as to whether, if that is true, we are allowed to do the thing we are asking to do.

The early tools built to handle this requirement, such as Sun's OpenSSO or Tivoli/IBM Access Manger, tended to involve exchanging a login/password for a random opaque token from an authorization server. They would then request a resource from another application server giving it this token in the request. That resource server, in turn, validates their token against the authorization server as well as retrieves additional information about the user, called `claims`, to decide if they are authorized. This process flow is illustrated by the diagram below (from OpenAM's documentation - which is a more recent opensource fork of Sun's OpenSSO):
![OAUTH2 Process Flow](/oauth2-ropc.svg)

This has approach been standardised and matured into the OAUTH then the OAUTH v2 standards. This standard is fairly complex and non-prescriptive though - leading to different tools often implementing differnet incompatbile subsets of the standard.

There are a few major challenges for availability and scaling of this approach. It means that nearly every request to a resource server depends on a subsequent validation request against an authorization server to succeed. That authorization server, in turn, depends on a database to keep track of which opaque tokens it has issued for whom, and which of those tokens are still valid, in order to respond to those validation requests. This led to rather complex and expensive architectures to ensure that the authorization servers are always available to handle all the requests such as the following - because if they go down or become very slow it can cause a major user-facing outage of your site/application.
![OpenAM HA Architecture](/active-active-deployment.png)

### JSON Web Tokens (JWT)

Originally, the token was not intended to have any inherent meaning to the clients using it - it was meant to be a random hexadecimal  that would be presetned to the authorization server for validation as well as retrieval any claims needed to process the request. However, the standard didn't say that it couldn't be something like JSON with the required data/claims right in it instead. This led to the development of JSON Web Tokens or JWTs as an alternative.

These JWTs not only embed the user details (claims) required by the resource server right in the token but then crytographically sign the whole thing with its private key (which becomes a sensitive secret in this model - anybody with this key can sign JWTs which the resource servers will trust!). This, together with an embedded expiry time, means that the process flow becomes much simpler as below:
![JWT Process Flow](/jwt.png)

Note that in this scenario the resource server does not need to talk to the authorization server at all - it just verifies the cryptographic signature in the token and makes sure the expiry date is in the future. And, since the majority of the load in the old model for the authorization server was around validating all the requests, this is a huge reduction in the load on the system. Also, the data around the token and whether it is valid is offloaded to each client/browser, and it's localData, rather than a centralised session state database backing the authorization server.

This approach does have one downside - once a JWT access token is issued under this model it is difficult to revoke before the expiry time embedded within it. This can be mitigated, though, by issuing tokens with short expiry times. Also, rather than having the user need to re-authenticate to get each new short-lived access token, many tools also allow for issuing a refresh token that can be exchanged for updated access tokens as required. And you can blacklist/revoke a refresh token so that the client can't refresh/renew any future access tokens from it and will need to re-authenticate (which they can't if you disable their access). As an example, if your access tokens are valid for one hour and your refresh tokens are valid for one day, and you want to revoke access to that user, you would blacklist their refresh token and disable subsequent logins so they would lose access in at most one hour when their current access token expires.

### OpenID Connnect

As we said earlier, the OATH v2 standard is not prescriptive enough to ensure that two different tools will necessarily be configured the same and be compatible. OpenID Connect is an extension or overlay on top of that standard to be more prescriptive and ensure compatability between providers and implementations. It also has the goal of helping to facilitate a single login to be used across several unrelated sites - for example logging into Spotify with your Facebook account.

It not only requires the use of JWTs instead of opaque tokens but outlines three types of JWTs - `id_token`, `access_token` and `refresh_token`.

* ID Token - contains claims about the identity of the authenticated user such as name, email, and phone_number.
* Access Token - grants access to authorized resources.
* Refresh Token - contains the information necessary to obtain a new ID or access token.

The standard splits access and id/claims into two different tokens because the info/claims from my Facebook account might not be relevant to Spotify in the example above - just that I have authenticated properly there and now have the relevant access token to present to Spotify.

## AWS Cognito

AWS has a service, `AWS Cognito`, to help with both authentication and authorization. It standards-based and adheres to the OAUTH v2 and OpenID Connect as well as SAML v2 (in order to federate with things such as Microsoft Active Directory).

As a fully managed service, it is easy to set up without any worries about standing up server infrastructure. It is an affordable pay-per-use service that has no minimums that can scale all the way from one to hundreds of millions of monthly active users (MAUs). It also integrates with many other AWS services such as API Gateway and Application Load Balancer which ensure that only authenticated requests are passed along to your services/APIs.

The two main components of Amazon Cognito are `User Pools` and `Identity Pools`. User pools are user directories that provide sign-up and sign-in options for your web and mobile app users. Identity pools provide AWS credentials to grant your users access to other AWS services. You can use user pools and identity pools either separately or together.

### User Pools

In order to authorioze users you need to have a directory of valid ones to compare those requests against. Cognito has the following options:

* It has a built-in user directory so you can simply store and manage your user details, including passwords, right in the Cognito User Pool. This is the default.
* It allows the optional use of external 'social' identity providers (IdPs) via OAUTH v2 such as Facebook, Google and Amazon. In this case Cognito delegates authentication of particular user(s) to these provider(s).
* It allows the optional use of SAML federation to identity providers (IdPs) such as a Microsoft Active Directory or Azure AD. In this case it delegates authentication of particular user(s) to these provider(s).

You can even leverage any/all of three options together to give your users options as to whether to use their social identities, their corporate identities or create a new application-specific one if you choose.

AWS User Pools generates JWT access tokens that can be validated by API Gateway, Application Load Balancer or by your own code (there are many libraries/packages to help with this). The first two are preferable because it saves you from needing to write authentication code and ensures that only authenticated users will ever reach your service. Your service will still need to implement authorization if different permissions between authenticated users is required.

### Identity Pools

The User Pools solves the "how to add a login/password to my app as-a-service" question - but it doesn't give you access to resources within AWS such as S3 buckets. That is what Identity Pools can do for you.

An Identity Pool doesn't have its own user directory or do its own authentication - is uses either a User Pool or any OAUTH v2 or SAML v2 identity provider (IdP). It is for authorization and exchanging an identity for a temporary AWS credential to use other AWS services/APIs.

## Walkthroughs of example web application secured by Cognito

There is a great example on AWS' github of the different ways you can use Cognito which we'll go through first - https://github.com/aws-samples/aws-cognito-apigw-angular-auth
![Cognito API Gateway Example](/cognito-apigw-example.png)
NOTE: This example currently is specifying a version of the Node.js Lambda runtime that has been deprecated - but if you bump it to nodejs8.10 then it will work. There is a pull request pending against the project to do that as well.

This example covers most of our exploration here - but it depends on API Gateway to validate/authorize your tokens or AWS role before passing the request on to your application. There are situations where you'll want to decode and authorize a JWT access token yourself - so we'll go through an example of how to do that too based on this example code from AWS' GitHub.

https://github.com/awslabs/aws-support-tools/tree/master/Cognito/decode-verify-jwt