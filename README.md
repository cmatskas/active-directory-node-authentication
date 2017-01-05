---
services: Azure Active Directory
platforms: node
author: msonecode
---

# How to authorize Node.js API with Azure AD

## Introduction
This is a web app sample with Node.js, and it demonstrates how to authorize a Node.js API with Azure AD.  
It uses the `adal-node` library to authorize with Azure AD.  
In this sample, you can test the authorized Node.js API with Azure AD.  

## Sample prerequisites
To open and run this sample, ensure that the following requisites have been met: 

- Node.js 6.6.0 or above.
- NPM is installed (it has to be installed before you installed Node.js).
- An active Azure subscription 
- You have the permission to manage Azure Active Directory on your subscription.

## Building the sample  
**Restore the libraries**  
- Open the Command Prompt window and navigate to the sample location folder. In this case, the sample location is D:\Sample\JSAzureADAuthWithNodeJS.  
  
  ![](README_IMAGE/image.png)

- Install the folling Node.js libraries:

    - `npm install express`
    - `npm install cookie-parser`
    - `npm install cookie-session`
    - `npm install crypto`
    - `npm install adal-node`

      ![](README_IMAGE/6c07f70c-7505-4764-bc87-b048dd5fe3f4image.png)

**Configuration App in Azure**

- Navigate to the [Azure Portal](https://portal.azure.com) and login.
- On the left hand side panel, find **Azure Active Directory** and click to open it. Alternatively, you can search for **Azure Active Directory** under **more services**.
  
  ![Open Active Directory in Azure Portal](README_IMAGE/a294ccd7-27dd-4837-85bb-0b71011e58f6image.png)

- Click the **App registrations** in Azure Active Directory management panel.
  
  ![App registrations](README_IMAGE/19f646df-59e3-48b7-84da-a37aabb60cfaimage.png)

- Click the **Add** button.
  
  ![Create new application](README_IMAGE/ae93830e-e252-4f98-a06f-1954b8a19ee9image.png)

- Fill in the **Name** and **Sign-on URL**, and choose the Application Type as **Web app / API**. Finally, click the **Create** button.
  
  ![Configure new application](README_IMAGE/e1994146-b800-46a8-ae72-fe07700d7850image.png)

- Once your application has been created and is visible in the application list, click to open it.
  
  ![Edit Application settings](README_IMAGE/302a25d2-76d0-4d97-a05f-586ce7984114image.png)

- You should be presented with the following blade. Click on **Settings**.
  
  ![Application Settings blade](README_IMAGE/5593875a-8154-446e-b43c-003aef4daa65image.png)

- Click on the **Reply URLs**, then change the URI address to [http://localhost:3000/getAToken](http://localhost:3000/getAToken). Finally, click on the **Save** button.
  
  ![Add Reploy URL](README_IMAGE/13a57657-68f7-4a17-889f-49602123b5eaimage.png)

- Click on **Keys**, then type a string in field **DESCRIPTION**, Choose EXPIRES as **Never expires**, type a string to the **VALUE** field, then click **Save** button.
  
  ![Create Application Key](README_IMAGE/ec70c194-0019-4b1e-8037-3f172e1af445image.png)

- IMPORTANT: After you have successfully created the key, copy its value. This is because it's only shown once and once you leave the blade you won’t be able to retrieve again.
  If you fail to copy it, you'll need to create a new key.

  ![Ensure you copy the key value](README_IMAGE/d8e30927-5da5-4a48-9992-077a734b83aeimage.png)

**Configuration parameters**  
With the Azure AD application in place we can now update to sample code to run the application. 
Open the `server.js` file and configure the following fields.

``` javascript
var authObj = require("./Auth.js").Create({
    tenant:"<your tenant name, e.g xxxx.onmicrosoft.com>",
    clientId:"<you appliction id in Azure AD>",
    secret:"<app key you copyied>",
    redirectUri:"http://localhost:3000/getAToken"
});

```

- **Tenant** This information can be found in the Azure Portal dashboard.
  
  ![Azure Tenant details](README_IMAGE/bba4fe89-0338-4623-b4dd-534b44dd1985image.png)

- **clientID** This information can be found in the portal under `Azure Active Directory -> App registrations -> <your application>`. Copy the **Application ID**.
  
  ![ClientId details](README_IMAGE/23e0b30d-0b1f-45ae-b112-16cf95a6f48cimage.png)

- **secret** This is the Azure Application key we created at the previous section.
  
  ![Application secret](README_IMAGE/ed957ee6-d5be-4f51-a14d-7ee62ac5da74image.png)

## Running the sample

- Open the Command Prompt window and navigate to the directory that the sample app code resides. For this example, the sample's location is D:\Sample\JSAzureADAuthWithNodeJS.
  
  ![Navigate to sample directory](README_IMAGE/e3f57e56-b93b-4563-aed1-c2a1d0f1b1f8image.png)

- Type command: **node server.js** to start the web server.
  
  ![Start the node server](README_IMAGE/fef5e68b-49cc-4156-967d-8fbf31198998image.png)

- Open the browser, and navigate to [http://localhost:3000](http://localhost:3000). Then click on the **Login** link.
  
  ![Open the browser and navigate to localhost:3000](README_IMAGE/f7783253-f923-4373-bc4d-18f587b0bbeeimage.png)

- Type your Azure credentials.
  
  ![Login with Azure credentials](README_IMAGE/caa7c331-d620-4ece-a800-8c7b2f033fb6image.png)

  > NOTE: this account must exist in your Azure AD.

- If the login is successful, you should be presented with the following page which contains all the authorization information.
  
  ![Authentication info successful](README_IMAGE/3d082ab6-a885-4f85-9004-7774342f4defimage.png)

## Using the code

**Auth.js**
``` javascript
'use strict';

var crypto = require('crypto');
var AuthenticationContext = require('adal-node').AuthenticationContext;

module.exports = {
    Create:function(params){
        var authObj = {
            tenant:params.tenant,
            clientId:params.clientId,
            secret:params.secret,
            redirectUri:params.redirectUri
        };

        authObj.authorityHostUrl = "https://login.windows.net";
        authObj.authorityUrl = authObj.authorityHostUrl + "/" + authObj.tenant;
        authObj.resource = "00000002-0000-0000-c000-000000000000";
        authObj.templateAuthzUrl = 'https://login.windows.net/' + authObj.tenant + '/oauth2/authorize?response_type=code&client_id=<client_id>&redirect_uri=<redirect_uri>&state=<state>&resource=<resource>';

        authObj.loginIfNotAuth = function(req,res,action){
            if(isAuthored(req))
            {
                if(isExpire(req))
                {
                    authObj.refreshToken(req,res,action);
                }
                else{
                    action();
                }
            }
            else
            {
                authWithAzureAD(res);
            }
        };

        authObj.receiveToken = function(req,res,action){
            if (req.cookies.authstate !== req.query.state) {
                res.send('error: state does not match');
                return;
            }

            var authenticationContext = new AuthenticationContext(authObj.authorityUrl);
            authenticationContext.acquireTokenWithAuthorizationCode(req.query.code, authObj.redirectUri, authObj.resource, authObj.clientId, authObj.secret, function(err, response) {
                var message = '';
                if (err) {
                    message = 'error: ' + err.message;
                    res.send(message);
                    return;
                }
                response.requestOn = Date.now();
                //set token to session
                req.session.authInfo = response;
                //do the action
                if(action){
                    action();
                }
            });
        };

        authObj.refreshToken = function(req,res,action) {
            var authenticationContext = new AuthenticationContext(authObj.authorityUrl);
            authenticationContext.acquireTokenWithRefreshToken(req.session.authInfo.refreshToken, authObj.clientId, authObj.secret, authObj.resource, function(refreshErr, refreshResponse) {
                if (refreshErr) {
                    var message = 'refreshError: ' + refreshErr.message;
                    res.send(message); 
                    return;
                }
                refreshResponse.requestOn = Date.now();
                //set token to session
                req.session.authInfo = refreshResponse;
                //do the action
                if(action){
                    action();
                }
            }); 
        };

        function authWithAzureAD(res){
            crypto.randomBytes(48, function(ex, buf) {
                var token = buf.toString('base64').replace(/\//g,'_').replace(/\+/g,'-');

                res.cookie('authstate', token);
                var authorizationUrl = createAuthorizationUrl(token);

                res.redirect(authorizationUrl);
            });
        }

        function isAuthored(req){
            return req.session.authInfo;
        }

        function isExpire(req){
            var now = Date.now();
            var requestOn = req.session.authInfo.requestOn;
            var expiresIn = req.session.authInfo.expiresIn * 1000;
            return requestOn + expiresIn >= Date.now();
        }

        function createAuthorizationUrl(state) {
            var authorizationUrl = authObj.templateAuthzUrl.replace('<client_id>', authObj.clientId)
                .replace('<redirect_uri>',authObj.redirectUri)
                .replace('<state>', state)
                .replace('<resource>', authObj.resource);
            return authorizationUrl;
        }

        return authObj;
    }
};

```

**Server.js**

``` javascript
'use strict';

var express = require('express');
var cookieParser = require('cookie-parser');
var cookieSession = require('cookie-session');

var authObj = require("./Auth.js").Create({
    tenant:"<your tenant name, e.g xxxx.onmicrosoft.com>",
    clientId:"<you appliction id in Azure AD>",
    secret:"<app key you copyied>",
    redirectUri:"http://localhost:3000/getAToken"
});

var app = express();
app.use(cookieParser('a deep secret'));
app.use(cookieSession({name: 'session',keys: [""]}));

app.get('/', function(req, res) {
    res.end('\
        <head>\
        <title>test</title>\
        </head>\
        <body>\
        <a href="./auth">Login</a>\
        </body>\
    ');
});

app.get('/auth', function(req, res) {
    authObj.loginIfNotAuth(req,res,function(){
        res.send("authed");
    });
});

app.get('/getAToken', function(req, res) {
    authObj.receiveToken(req,res,function(){
        res.redirect('/AuthInfo');
    });
});

app.get('/AuthInfo', function(req, res) {
    var sessionValue = req.session.authInfo;
    var authString = JSON.stringify(sessionValue);
    var userID = sessionValue.userId;
    var familyName = sessionValue.familyName;
    var givenName = sessionValue.givenName;

    res.end(`\
        <h1>UserID: ${userID}</h1>
        <h2>familyName: ${familyName}</h2>
        <h2>givenName: ${givenName}</h2>
        <h2>full data:</h2>
        <p>${authString}</p>
    `);
});

app.listen(3000);
console.log("listen 3000");
```

## More information
- Windows Azure Active Directory Authentication Library (ADAL) for Node.js
[https://github.com/AzureAD/azure-activedirectory-library-for-nodejs](https://github.com/AzureAD/azure-activedirectory-library-for-nodejs)