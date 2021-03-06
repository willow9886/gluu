# Gluu Oxd Server Tutorial

In this tutorial I am going to explain how we can interact with Gluu oxd Server for SSO with the [authorization code flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) 
using Python CGI and Apache, in the hopes that programmers of other languages will benefit.

## Preliminary

For this tutorial we need a Gluu Server 3.1.4, Gluu oxd Server 4.0.beta and a https & cgi enabled
web server, Apache in our case. 

### Gluu Server 3.1.4 (OP)
Gluu Server will act as the OpenID Connect Provider (OP). Follow 
[these instructions](https://gluu.org/docs/ce/3.1.4/installation-guide/install/) 
to install your Gluu Server. In this tutorial I installed Gluu Server on host **op.server.com**
Add a test user. I added user `test_user`

### Gluu oxd Server 4.0.beta
To install Gluu oxd Server 4.0, follow 
[these instructions](https://github.com/GluuFederation/oxd/wiki/oxd-4.0.beta). Gluu oxd Server will be installed on host **oxd.server.com**, but could be installed on the same server as the Gluu Server if needed, as there will be no port conflicts.

My **defaultSiteConfig** section of `oxd-server.yml` configuration is as follows:

```
defaultSiteConfig:
  op_host: 'https://op.server.org'
  op_discovery_path: ''
  response_types: ['code']
  grant_type: ['authorization_code']
  acr_values: ['']
  scope: ['openid', 'profile', 'email']
  ui_locales: ['en']
  claims_locales: ['en']
  contacts: []
```

### Apache Web Server (RP)

Since we are going to write a cgi script for simplicity, we first need to get a working
web server to act as the Relying Party (RP). Apache will be installed on the host **rp.server.com**. I am using Ubuntu 16.04 LTS for this purpose. First install apache web
server:

```
sudo apt-get update
sudo apt-get install apache2
```

Enable ssl, cgi and https:

```
sudo a2enmod cgi
sudo a2enmod ssl
sudo a2ensite default-ssl.conf
```

We will use Python's requests module to interact with oxd's REST API:

```
sudo apt-get install python-requests
```

## Steps for SSO with Gluu oxd Server

These are the steps that will be performed for SSO, step numbers maps the steps in cgi script

Step | Explanation | Endpoint
-----|-------------|----------
1 | Creating a client on Gluu server and registering your site to oxd Server  (we will do this dynamically, for clarities sake, though it is possible to automate this step in the cgi script). | [register-site](https://gluu.org/docs/oxd/api/#register-site)
2 | Obtain access token from oxd Server. This token will be used in headers to authenticate to oxd server in all subsequent queries. I will call this as `oxd_access_token`. So, with the exception of this step headers of subsequent queries will be: <br> `Content-type: 'application/json` <br> `Authorization': 'Bearer <oxd_access_token>'` | [get-client-token](https://gluu.org/docs/oxd/api/#get-client-token)
3 | Get authorization url. User will click on this url to reach Gluu's login page and will will be redirected to `authorization_redirect_uri` | [get-authorization-url](https://gluu.org/docs/oxd/api/#get-authorization-url)
4 | We need to verify if `code` and `state` values returned by browser to our cgi script after authorization by Gluu Server. We can pass these values to oxd Server and obtain an access token to user's claims. | [get-tokens-by-code](https://gluu.org/docs/oxd/api/#get-tokens-id-access-by-code)
5 | Query user information and display on page. | [get-user-info](https://gluu.org/docs/oxd/api/#get-user-info)
6 | Query logout uri from oxd server. | [get-logout-uri](https://gluu.org/docs/oxd/api/#get-logout-uri)

## Creating Client and Registering Site to Oxd Server

Before start working on oxd-server, we need two settings on Gluu Server

* Enable dynamic registration of clients: **Configuration->Manage Custom Script**, 
 click on **Client Registration** tab and and enable `client_registration` script, and
 click **Update** button. If you don't want to enable dynamic client registration,
 please register [a client manually](create_client.md).
 
* Enable dynamic registration of "profile" scope: **OpenID Connect->Scopes**,
  click on the `profile` scope and set `Allow for dynamic registration` to "True".
  clieck **Update** button.

Following through first step we will register the client dynamically, and register the site to oxd. Write the following content to `data.json`:

```
{
  "authorization_redirect_uri": "https://rp.server.com/cgi-bin/oxd.py/login",
  "op_host": "https://op.server.com",
  "post_logout_redirect_uri": "https://rp.server.com/cgi-bin/oxd.py/logout",
  "application_type": "web",
  "response_types": ["code"],
  "grant_types": ["authorization_code", "client_credentials"],
  "scope": ["openid", "oxd", "profile"],
  "acr_values": [""],
  "client_name": "TestRPClient",
  "client_jwks_uri": "",
  "client_token_endpoint_auth_method": "",
  "client_request_uris": [""],
  "client_frontchannel_logout_uris": [""],
  "client_sector_identifier_uri": "",
  "contacts": ["admin@example.com"],
  "redirect_uris": [],
  "ui_locales": [""],
  "claims_locales": [""],
  "claims_redirect_uri": [],
  "client_id": "",
  "client_secret": "",
  "trusted_client": true
}
```

!!! Note
    If you created client manually you must supply `client_id` and `client_secret`

Now we create the client and regsiter our site on oxd server:

```
curl -k -X POST https://oxd.server.com:8443/register-site --header "Content-Type: application/json" -d @data.json 

```

provide full path to `data.json` or execute this command in the same directory. This will output as follows:

```
{
  "client_id_issued_at": 1540819675,
  "client_registration_client_uri": "https://op.server.com/oxauth/restv1/register?client_id=@!5856.8A6C.09D4.C454!0001!655A.E96C!0008!D619.DF5D.B48E.DF10",
  "client_registration_access_token": "08ed04b0-6207-4d53-8d51-523cc3bf5c19",
  "oxd_id": "6179a64f-bfe3-44b3-85b5-c01f86336ef7",
  "client_id": "@!5856.8A6C.09D4.C454!0001!655A.E96C!0008!D619.DF5D.B48E.DF10",
  "client_secret": "d384ec9b-00a0-46cf-a92f-fea1225b0717",
  "op_host": "https://op.server.com",
  "client_secret_expires_at": 1540906075
}
```

Take a note of `oxd_id`, `client_secret` and  `client_secret`, we will use them in subsequent queries and configuration. Be aware that these are uniquely generated identifiers and will not match what is shown above.

CGI Script
----------
On the RP server (Apache/HTTPD), write the following content to `/usr/lib/cgi-bin/oxd.py` and make it executable with
`chmod +x /usr/lib/cgi-bin/oxd.py`

```
#!/usr/bin/python

import requests
import os
import json
import cgi

oxd_id = '$YOUR_OXD_ID'
client_secret = '$YOUR_CLIENT_SECRET'
client_id = '$YOUR_CLIENT_ID'
op_host = 'https://op.server.com/'
oxd_server = 'https://oxd.server.com:8443/'

print 'Content-type: text/html'


def post_data(end_point, data, access_token):
    """Posts data to oxd server"""
    headers = {
                'Content-type': 'application/json', 
                'Authorization': "Bearer " + access_token
        }

    result = requests.post(
                    oxd_server + end_point, 
                    data=json.dumps(data), 
                    headers=headers, 
                    verify=False
                    )

    return result.json()

# We will use PATH_INFO which URI is requested
path_info = os.environ.get('PATH_INFO','/')

#If login is requested
if path_info.startswith('/login'):
    print
    args = cgi.parse_qs(os.environ[ "QUERY_STRING" ])
    
    # Read access_token that we previously saved
    oxd_access_token = open('/tmp/oxd_access_token.txt').read()

    data = {
        "oxd_id": oxd_id,
        "code": args['code'][0],
        "state": args['state'][0]
    }

    # [4] Request access token to retreive user info. If you don't need user info
    # It is enough to make sure the user logged in
    result = post_data('get-tokens-by-code', data, oxd_access_token)

    data = {
        "oxd_id": oxd_id,
        "access_token": result['access_token']
    }

    # [5] Now we can retreive user information.
    result = post_data('get-user-info', data, oxd_access_token)
    
    # Finally print user info
    for cl in result['claims']:
        print '<br><b>{0} :</b> {1}'.format(cl, result['claims'][cl][0])
    
    print '<br><a href="/logoutme">Click here to logout</a>'

#If user wants to logout, he should first come to this pasge
elif path_info.startswith('/logoutme'):

    data = {
        "oxd_id": oxd_id,
    }

    # Read access_token that we previously saved
    oxd_access_token = open('/tmp/oxd_access_token.txt').read()

    # [6] Request logout uri from oxd server.
    result = post_data('get-logout-uri', data, oxd_access_token)

    
    # print redirection headers
    print "Status: 303 Redirecting"
    print "Location: "+ result['uri']
    print

# User will be redirected after logging out
elif path_info.startswith('/logout'):
    print
    print "logged out"

else:
    print
    # [2] We need to get access_token from oxd-server
    data = {
      "op_host": op_host,
      "scope": ["openid", "oxd", "profile", "email"],
      "client_id": client_id,
      "client_secret": client_secret,
      "authentication_method": "",
      "algorithm": "",
      "key_id": ""
    }

    result = post_data('get-client-token', data, '')
    oxd_access_token = result['access_token']

    # Since cgi scripts runs per request, we need access_token in subsequent queries.
    # Thus write a file. access_token should be secured and better to use session
    # in web frameworks.
    with open('/tmp/oxd_access_token.txt','w') as W:
        W.write(oxd_access_token)

    data = {
      "oxd_id": oxd_id,
      "scope": ["openid", "profile", "email"],
      "acr_values": [],
    }

    # [3] User will be directed to Gluu Server to login, so we need an url for login
    result = post_data('get-authorization-url', data, oxd_access_token)

    print '<a href="{0}">Click here to login</a>'.format(result['authorization_url'])
```

Now navigate to https://rp.server.com/cgi-bin/oxd.py you will see link **Click here to login**
as below

![login link](fig1.png)

Click the link, you will be redirected to Gluu login page, after entering credidentals and allowing
RP to access claims, you will see user info as:

![user info](fig2.png)
