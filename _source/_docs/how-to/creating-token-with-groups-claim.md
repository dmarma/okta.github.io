---
layout: docs_page
title: Create a Token with a Groups Claim Using the App Profile
excerpt: How to use the app profile to create an ID token or access token that contains a groups claim
---

## How to Create a Token with a Groups Claim Using the App Profile

You can add a groups claim for any combination of application groups and user groups into ID tokens to perform SSO using Okta Authorization Server, or ID tokens and access tokens to perform authentication and authorization using Custom Authorization Server (API Access Management required).
This process optionally uses Okta's flexible app profile, which accepts any JSON-compliant content, to create a whitelist of groups
that can then easily be referenced. This is especially useful if you have a large number of groups to whitelist or otherwise
need to set group whitelists on a per-application basis.

### Before You Start

Perform the following two tasks before you start for either Okta Authorization Server or Custom Authorization Server.

 * Create an OAuth 2.0 or OpenID Connect client with the [Apps API](/docs/api/resources/apps#request-example-8). In the instruction examples, the client ID is `0oabskvc6442nkvQO0h7`.
 * Create the groups that you wish to configure in the groups claim. In the instruction examples, we're configuring the `WestCoastDivision` group, and the group ID is `00gbso71miOMjxHRW0h7`.

Now use the instructions for your chosen authorization server to create a groups claim, assign a group whitelist to your client app, and configure a groups claim that references the whitelist:

* [Create a Token with Groups Claim and Okta Authorization Server](#create-a-token-with-groups-claim-okta-authorization-server)
* [Create a Token with Groups Claim and Custom Authorization Server](#create-a-token-with-groups-claim-custom-authorization-server)

### Create a Token with Groups Claim (Okta Authorization Server)

Use this procedure if you have the Okta Authorization Server, not the `default` or other Custom Authorization Server.

#### Step One: Get the Group IDs

Send a request to `https://{yourOktaDomain}.com/api/v1/groups` and collect the IDs for all the groups you'll want to whitelist.

   Request Example:

~~~sh
curl -X GET \
  -H 'accept: application/json' \
  -H 'authorization: SSWS ${api_token}' \
  -H 'content-type: application/json' \
  "https://{yourOktaDomain}.com/api/v1/groups"
~~~

   Response Example:

~~~json
[
  {
    "id": "00gbso71miOMjxHRW0h7",
    "created": "2017-08-25T21:15:48.000Z",
    "lastUpdated": "2017-08-25T21:15:48.000Z",
    "lastMembershipUpdated": "2017-08-25T21:16:07.000Z",
    "objectClass": [
      "okta:user_group"
    ],
    "type": "OKTA_GROUP",
    "profile": {
      "name": "WestCoastDivision",
      "description": "Employees West of the Rockies"
    },
    "_links": {
      "logo": [
        {
          "name": "medium",
          "href": "https://op1static.oktacdn.com/assets/img/logos/groups/okta-medium.d7fb831bc4e7e1a5d8bd35dfaf405d9e.png",
          "type": "image/png"
        },
        {
          "name": "large",
          "href": "https://op1static.oktacdn.com/assets/img/logos/groups/okta-large.511fcb0de9da185b52589cb14d581c2c.png",
          "type": "image/png"
        }
      ],
      "users": {
        "href": "https://{yourOktaDomain}.com/api/v1/groups/00gbso71miOMjxHRW0h7/users"
      },
      "apps": {
        "href": "https://{yourOktaDomain}.com/api/v1/groups/00gbso71miOMjxHRW0h7/apps"
      }
    }
  }
]
~~~

This example uses one `groupId` for simplicity's sake.

#### Step Two: Add List of Groups to Profile of Client App

If you only have one or two groups to specify, simply add the group IDs to the first parameter of the `getFilteredGroups` function described in the next step.
However, if you have a lot of groups to whitelist, you can put the group IDs in the client app's profile property bag: `https://{yourOktaDomain}.com/api/v1/apps/${applicationId}`.
The following example names the group whitelist `groupwhitelist`, but you can name it anything.

Request Example:

~~~sh
curl -X PUT \
  https://{yourOktaDomain}.com/api/v1/apps/0oabskvc6442nkvQO0h7 \
  -H 'accept: application/json' \
  -H 'authorization: SSWS ${api_token}' \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
    "name": "oidc_client",
    "label": "Sample Client",
    "status": "ACTIVE",
    "signOnMode": "OPENID_CONNECT",
    "credentials": {
      "oauthClient": {
        "token_endpoint_auth_method": "client_secret_post"
      }
    },
    "settings": {
      "oauthClient": {
        "client_uri": "http://localhost:8080",
        "logo_uri": "http://developer.okta.com/assets/images/logo-new.png",
        "redirect_uris": [
          "https://example.com/oauth2/callback",
          "myapp://callback"
        ],
        "response_types": [
          "token",
          "id_token",
          "code"
        ],
        "grant_types": [
          "implicit",
          "authorization_code"
        ],
        "application_type": "native"
      }
    },
    "profile": {
      "groupwhitelist": [
        "00gbso71miOMjxHRW0h7"
      ]
    }
  }'
~~~

You can add application groups, user groups, or both to the group whitelist, specified as an array of IDs.

To use the group whitelist for every client that gets this claim in a token, put the attribute name of the whitelist in the first parameter of the `getFilteredGroups` function described in the next step.

#### Step Three: Configure a Custom Claim for Your Groups (Okta Authorization Server)

For the Okta Authorization Server, you can only create an ID token with a groups claim, not an access token.

> This step requires the Okta user interface. If you are using the Developer Console, select the drop-down control in the left side of the top banner to switch to Classic UI for Step Three.

 a. In the Okta user interface, navigate to the Sign On tab of the client application you are configuring, and click the **Edit** button in the Open ID Connect ID Token section.

 b. In **Groups claim type**, choose **Expression**.

 c. In **Group claims filter**, leave the default name `groups` or change it if you wish.

 d. In **Groups claim expression**, add this expression: `getFilteredGroups(app.profile.groupwhitelist, "group.name", 40)`.

#### Step Four: Send a Test Request

To obtain a token with the configured groups claim, send a request for an ID token that includes the `groups` claim set in step 3.c. as a scope to `https://{yourOktaDomain}.com/oauth2/v1/authorize`, as illustrated in the following example.

Request Example for Okta Authorization Server:

~~~sh
curl -X GET \
  "https://{yourOktaDomain}.com/oauth2/v1/authorize?client_id=0oabskvc6442nkvQO0h7
  &response_type=id_token
  &response_mode=fragment&scope=openid%20groups
  &redirect_uri=https%3A%2F%2Fexample.com
  &state=myState&nonce=${yourNonceValue}' \
~~~

#### Step Five: Decode the JWT to Verify

Decode the JWT in the response to see that the groups are in the token. For example, this JWK contains the `groups` claim in an ID token:

 ~~~JSON
eyJraWQiOiJiS0U0czM3d01tQWZ5ZzQtVFJQcVg1YW50blE1ajBuNFJKcE9nSl9zT0JVIiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiIwMHU1dDYwaWxvT0hOOXBCaTBoNyIsInZlciI6MSwiaXNzIjoiaHR0cHM6Ly9teXN0aWNvcnAub2t0YXByZXZpZXcuY29tIiwiYXVkIjoiMG9hYnNrdmM2NDQybmt2UU8waDciLCJpYXQiOjE1MTQ0OTc3ODEsImV4cCI6MTUxNDUwMTM4MSwianRpIjoiSUQua0FlMWFzU08wM00walp0Y2ZHZGtpWGwwUW9LRHE5aHl3OE1VUU1UNGwtWSIsImFtciI6WyJwd2QiXSwiaWRwIjoiMDBvNXQ2MGlsM1V6eUllNXYwaDciLCJub25jZSI6IjQ1YzExMDJiLTM0MmUtNGZjMC04ZDllLWM0NTY0MmFlOWFkOCIsImF1dGhfdGltZSI6MTUxNDQ5NjM1MCwiZ3JvdXBzIjpbIldlc3RDb2FzdERpdmlzaW9uIl19.ACKbJZ-lbGtgBAQDhamq7K9WJzHS0WySN0R2LXSkBahWWVMU1W-oTh2xDuHmyQv6HZpk-V4epnk-OItRBQb214NsRG8AJGn5n3QGYp5xPWVXXQ_hFZSro4br6Rdn_U8iZebqs6EXpGhxG7tN9VEgB-SkAynHdy2MbQpikGWcxORSA8vQLQhDRt2VZDobienTA8zLeThzOyAmhPjELxHRHFVT1OOrEoCqUV6wlk8LfhATRlxZGm6lrlZQbqxV_PDM8u7zN0l9XV01Rh0WHO7zZ_Oq0PEeQkf-TC9x7Gl_pOuRyRfGEsrqq-ZEL6AZszxotRKQJO1nNahAhfbNESO2mg
~~~

Example Payload Data for ID Token:

~~~JSON
{
  "sub": "00u5t60iloOHN9pBi0h7",
  "ver": 1,
  "iss": "https://${yourOktaDomain}.com",
  "aud": "0oabskvc6442nkvQO0h7",
  "iat": 1514497781,
  "exp": 1514501381,
  "jti": "ID.kAe1asSO03M0jZtcfGdkiXl0QoKDq9hyw8MUQMT4l-Y",
  "amr": [
    "pwd"
  ],
  "idp": "00o5t60il3UzyIe5v0h7",
  "nonce": "${yourNonceValue}",
  "auth_time": 1514496350,
  "groups": [
    "WestCoastDivision"
  ]
}
~~~

The ID token contains the group WestCoastDivision so the audience (`aud`) has access to the group information about the user.

For flows other than implicit, post to the token endpoint `https://{yourOktaDomain}.com/oauth2/v1/token` with the user or client you want. Make sure the user is assigned to the app and to one of the groups from your whitelist.

If the results aren't as expected, start your troubleshooting by inspecting the System Log to see what went wrong. Also, try requesting only an ID token instead of both and ID token and access token.

### Create a Token with Groups Claim (Custom Authorization Server)

#### Step One: Get the Group IDs

Send a request to `https://{yourOktaDomain}.com/api/v1/groups` and collect the IDs for all the groups you'll want to whitelist.

Request Example:

~~~sh
curl -X GET \
  -H 'accept: application/json' \
  -H 'authorization: SSWS ${api_token}' \
  -H 'content-type: application/json' \
  "https://{yourOktaDomain}.com/api/v1/groups"
~~~

This example uses one `groupId` for simplicity's sake.

 Response Example:

~~~json
[
  {
    "id": "00gbso71miOMjxHRW0h7",
    "created": "2017-08-25T21:15:48.000Z",
    "lastUpdated": "2017-08-25T21:15:48.000Z",
    "lastMembershipUpdated": "2017-08-25T21:16:07.000Z",
    "objectClass": [
      "okta:user_group"
    ],
    "type": "OKTA_GROUP",
    "profile": {
      "name": "WestCoastDivision",
      "description": "Employees West of the Rockies"
    },
    "_links": {
      "logo": [
        {
          "name": "medium",
          "href": "https://op1static.oktacdn.com/assets/img/logos/groups/okta-medium.d7fb831bc4e7e1a5d8bd35dfaf405d9e.png",
          "type": "image/png"
        },
        {
          "name": "large",
          "href": "https://op1static.oktacdn.com/assets/img/logos/groups/okta-large.511fcb0de9da185b52589cb14d581c2c.png",
          "type": "image/png"
        }
      ],
      "users": {
        "href": "https://{yourOktaDomain}.com/api/v1/groups/00gbso71miOMjxHRW0h7/users"
      },
      "apps": {
        "href": "https://{yourOktaDomain}.com/api/v1/groups/00gbso71miOMjxHRW0h7/apps"
      }
    }
  }
]
~~~

#### Step Two:  Add List of Groups to Profile of Client App

If you only have one or two groups to specify, simply add the group IDs to the first parameter of the `getFilteredGroups` function described in the next step.
However, if you have a lot of groups to whitelist, you can put the group IDs in the client app's profile property bag: `https://{yourOktaDomain}.com/api/v1/apps/${applicationId}`.

This example names the group whitelist `groupwhitelist`, but you can name it anything.

Request Example:

~~~sh
curl -X POST \
  https://{yourOktaDomain}.com/api/v1/apps/0oabskvc6442nkvQO0h7 \
  -H 'accept: application/json' \
  -H 'authorization: SSWS ${api_token}' \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
    "name": "oidc_client",
    "label": "Sample Client",
    "status": "ACTIVE",
    "signOnMode": "OPENID_CONNECT",
    "credentials": {
      "oauthClient": {
        "token_endpoint_auth_method": "client_secret_post"
      }
    },
    "settings": {
      "oauthClient": {
        "client_uri": "http://localhost:8080",
        "logo_uri": "http://developer.okta.com/assets/images/logo-new.png",
        "redirect_uris": [
          "https://example.com/oauth2/callback",
          "myapp://callback"
        ],
        "response_types": [
          "token",
          "id_token",
          "code"
        ],
        "grant_types": [
          "implicit",
          "authorization_code"
        ],
        "application_type": "native"
      }
    },
    "profile": {
      "groupwhitelist": [
        "00gbso71miOMjxHRW0h7"
      ]
    }
  }'
~~~

You can add application groups, user groups or both to the group whitelist, specified as an array of IDs.

To use the group whitelist for every client that gets this claim in a token, put the attribute name of the whitelist in the first parameter of the `getFilteredGroups` function described in the next step.

#### Step Three: Configure a Custom Claim for Your Groups (Custom Authorization Server)

Add a custom claim for the ID token or access token on a Custom Authorization Server with the following function:

~~~
 getFilteredGroups({app.profile.whitelist},
 "{value-to-represent-the-group-in-the-token}",
 {maximum-number-of-groups-to-include-in-token})
~~~

The maximum number of groups specified must be less than 100.

In the following two examples for creating the groups claim, the `name` for the claim is `groups`, but you can name it whatever you wish.

##### Request Example for Access Token

~~~sh
curl -X POST \
  https://{yourOktaDomain}.com/api/v1/authorizationServers/ausain6z9zIedDCxB0h7/claims \
  -H 'accept: application/json' \
  -H 'authorization: SSWS ${api_token}' \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
   "name": "groups",
   "status": "ACTIVE",
   "claimType": "RESOURCE",
   "valueType": "EXPRESSION",
   "value": "\"getFilteredGroups(app.profile.groupwhitelist, \"group.name\", 40)\"",
   "conditions": {
     "scopes": []
   }
 }'
~~~

##### Request Example for ID Token

~~~sh
curl -X POST \
  https://{yourOktaDomain}.com/api/v1/authorizationServers/ausain6z9zIedDCxB0h7/claims \
  -H 'accept: application/json' \
  -H 'authorization: SSWS ${api_token}' \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/json' \
  -d '{
   "name": "groups",
   "status": "ACTIVE",
   "claimType": "IDENTITY",
   "valueType": "EXPRESSION",
   "value": "\"getFilteredGroups(app.profile.groupwhitelist, \"group.name\", 40)\"",
   "conditions": {
     "scopes": []
   }
 }'
 ~~~

Be sure that you have a policy and rule set up in your Custom Authorization Server or the request in the next step won't work.

See [group function documentation](/reference/okta_expression_language/#group-functions) for more information about specifying groups with `getFilteredGroups`.

Now when you mint a token, groups in the `groupwhitelist` that also have the user as a member are included in the `groups` claim. Test your configuration in the next step.

#### Step Four: Send a Test Request

To obtain a token with the configured groups claim, send a request for an ID token that includes one of the scopes that the claim is associated with: `https://{yourOktaDomain}.com/oauth2/${authorizationServerId}/v1/authorize`.

Request Example for Custom Authorization Server:

~~~sh
 curl -X GET \
  'https://${yourOktaDomain}.com/oauth2/ausain6z9zIedDCxB0h7/v1/authorize?client_id=0oabskvc6442nkvQO0h7
    &response_type=id_token
    &response_mode=fragment
    &scope=groups%20openid
    &redirect_uri=https%3A%2F%2Fexample.com
    &state=myState&nonce=${myNonceValue}' \
 ~~~

>Note:
* In this example, the claim was configured to work with all scopes. If you specify only certain scopes to return the claim, you'll need to specify one of them in the request.
* To obtain an access token, simply change `response_type=id_token` to `response_type='token'`.

#### Step Five: Decode the JWT to Verify

Decode the JWT in the response to see that the groups are in the token. For example, this JWK contains the group claim:

~~~JSON
eyJraWQiOiJ2U2N0OVJ0R2g5ang5QVFfT05aNEFhM19lZ3YwVlktelJKWTZvbmE5R3o4IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiIwMHU1dDYwaWxvT0hOOXBCaTBoNyIsInZlciI6MSwiaXNzIjoiaHR0cHM6Ly9teXN0aWNvcnAub2t0YXByZXZpZXcuY29tL29hdXRoMi9hdXNhaW42ejl6SWVkREN4QjBoNyIsImF1ZCI6IjBvYWJza3ZjNjQ0Mm5rdlFPMGg3IiwiaWF0IjoxNTE0NDk3MzQ2LCJleHAiOjE1MTQ1MDA5NDYsImp0aSI6IklELlo1RkhPY1BhUUI3Q3ExeXhkRElIRzhieDd0Y3gxVGxYcFdvZTY4cHpGSDgiLCJhbXIiOlsicHdkIl0sImlkcCI6IjAwbzV0NjBpbDNVenlJZTV2MGg3Iiwibm9uY2UiOiIzODBiNTgwMS05MTYzLTRjNGEtOWMyMS1kNjBhMmEzMzJhNzciLCJhdXRoX3RpbWUiOjE1MTQ0OTYzNTAsIm15Z3JvdXBXaGl0ZWxpc3QiOlsiV2VzdENvYXN0RGl2aXNpb24iXX0.X4_rs_bgmWW5cX6p-fur_EN4-Uf2hz3jZZVUgdBRUX0x64O7wbmuPXGicjfLIMH6HRx7bETPjALNoSjvUrFI1IEHBMVROZQGvAYtB5f5ge6ZvZVNk0B8Coz6h3Y9vLmZGwxOFHR0_bbQQC2j01wKKeFPjznfMxtEuBLkD2DXuF7WkHZSmMG5dp7L9LUpvwfCQ2fv1SYRQ_pRVGIxZK5jh9O2yip4LMANbayDkF0Ud8lbq9CAv3Zz4tG77Cwou87yphnHlPgHDrCRRiEbCoe6Q1l8UIfMC3kfaT2HoyJb6jvA91h89jgRbIvUEfasrLoSwUJQv-sYz302QiQdF8WZAQ
~~~

Example Payload Data for an ID Token:

~~~JSON
{
  "sub": "00u5t60iloOHN9pBi0h7",
  "ver": 1,
  "iss": "https://{yourOktaDomain}.com/oauth2/ausain6z9zIedDCxB0h7",
  "aud": "0oabskvc6442nkvQO0h7",
  "iat": 1514497346,
  "exp": 1514500946,
  "jti": "ID.Z5FHOcPaQB7Cq1yxdDIHG8bx7tcx1TlXpWoe68pzFH8",
  "amr": [
    "pwd"
  ],
  "idp": "00o5t60il3UzyIe5v0h7",
  "nonce": "${myNonceValue}",
  "auth_time": 1514496350,
  "groups": [
    "WestCoastDivision"
  ]
}
~~~

Example Payload Data for an Access Token:

~~~JSON
{
  "aud": "https://{yourOktaDomain}",
  "sub": "annie.jackson@acme.com",
  "iat": 1511983934,
  "exp": 1511987534,
  "cid": "0oabskvc6442nkvQO0h7",
  "uid": "00u5t60iloOHN9pBi0h7",
  "scp": [
    "openid"
  ],
  "groups": [
    "WestCoastDivision"
  ]
}
~~~

The ID token or access token contains the group WestCoastDivision so the audience (`aud`) has access to the group information about the user.

For flows other than implicit, post to the token endpoint `https://{yourOktaDomain}.com/oauth2/${authorizationServerId}/v1/token` with the user or client you want. Make sure the user is assigned to the app and to one of the groups from your whitelist.

If the results aren't as expected, start your troubleshooting by inspecting the System Log to see what went wrong. Also, try requesting only an ID token instead of both and ID token and access token.