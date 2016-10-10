---
services: Azure Active Directory
platforms: node
author: msonecode
---

# How to authorize Node.js API with Azure AD

## Introduction
This is a web app sample with Node.js, and it demonstrates how to authorize Node.js API with Azure AD,  
It uses the adal-node library to authorize with Azure AD.  
In this sample, you can test the authorized Node.js API with Azure AD.  

## Sample prerequisites
To open and run this asmple, ensure that the following requisites have been met:
- Node.js 6.6.0 or above.
- NPM is installed (In default case, it has to be installed before you installed Node.js).
- You have subcribed the Azure and you have the permission to manage Azure Active Directory on your subscription.

## Building the sample  
**Restore the library**  
- Open the Command Prompt window and navigate to sample location folder. In this case, the sample location is D:\Sample\JSAzureADAuthWithNodeJS.  
  ![][1]
- Install the libraries  
    - **npm install express**
    - **npm install cookie-parser**
    - **npm install cookie-session**
    - **npm install crypto**
    - **npm install adal-node**
      ![][2]

**Configuration App in Azure**
- Go to [https://code.msdn.microsoft.com/How-to-authorize-Nodejs-fdc580ed/https://portal.azure.com](https://code.msdn.microsoft.com/How-to-authorize-Nodejs-fdc580ed/https://portal.azure.com) and login.
- In the left panel, find **Azure Active Directory** and click, or click **more services** to find the **Azure Active Directory** in prop panel.
  ![][3]

- Click the App registrations in manage panel from Azure Active Directory.
  ![][4]

- Click the **add** button.
  ![][5]

- Fill in the **Name** and **Sign-on URL**, and choose the Application Type as **Web app / API**. Finally, click **Create** button.
  ![][6]

- Then you can notice the app has been created and appeared in the app list, click to open it.
  ![][7]

- After that, you can see the below UI, click **Settings**.
  ![][8]

- Click **Reply URLs**, then change address as [http://localhost:3000/getAToken](http://localhost:3000/getAToken). Finally, click the **Save** button.
  ![][9]

- Click **Keys**, then type a string in field **DESCRIPTION**, Choose EXPIRES as **Never expires**, type a string to the **VALUE** field, then click **Save** button.
  ![][10]  
  After you have successfully created the key, copy its valuedown. Since it shows only once, you won’t be able to retrieve again after you leave this blade.  
  ![][11]

**Configuration parameters**  
Open the file **server.js** and configuration flowing field.
![][12]
- **Tenant**  
  You can find it in dashboard.
  ![][13]
- **clientID**
  **Azure Active Directory** -> **Appregistrations** -> **application of you created** -> copy the **Application ID** down.
  ![][14]

- **secret**
  **application of you created** -> **Settings** -> **Keys** -> **you copied value**.
  ![][15]

## Running the sample
- Open the Command Prompt window and navigate to sample location folder, in this case, the sample location is D:\Sample\JSAzureADAuthWithNodeJS.
  ![][16]

- Type command: **node server.js** so that the web server will be running.
  ![][17]

- Open the browser, and go to [http://localhost:3000](http://localhost:3000), click the **Login** link.
  ![][18]

- Type your Microsoft account info. Note: this account must be the member in your Azure AD.
  ![][19]

- After the authorization has been finished, you can see the below page which contains all the authorization information.
  ![][20]

## Using the code
**Auth.js**
```javascript
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
  [https://code.msdn.microsoft.com/How-to-authorize-Nodejs-fdc580ed/https://github.com/AzureAD/azure-activedirectory-library-for-nodejs](https://code.msdn.microsoft.com/How-to-authorize-Nodejs-fdc580ed/https://github.com/AzureAD/azure-activedirectory-library-for-nodejs)

[1]: README_IMAGE/image.png
[2]: README_IMAGE/6c07f70c-7505-4764-bc87-b048dd5fe3f4image.png
[3]: README_IMAGE/a294ccd7-27dd-4837-85bb-0b71011e58f6image.png
[4]: README_IMAGE/19f646df-59e3-48b7-84da-a37aabb60cfaimage.png
[5]: README_IMAGE/ae93830e-e252-4f98-a06f-1954b8a19ee9image.png
[6]: README_IMAGE/e1994146-b800-46a8-ae72-fe07700d7850image.png
[7]: README_IMAGE/302a25d2-76d0-4d97-a05f-586ce7984114image.png
[8]: README_IMAGE/5593875a-8154-446e-b43c-003aef4daa65image.png
[9]: README_IMAGE/13a57657-68f7-4a17-889f-49602123b5eaimage.png
[10]: README_IMAGE/ec70c194-0019-4b1e-8037-3f172e1af445image.png
[11]: README_IMAGE/d8e30927-5da5-4a48-9992-077a734b83aeimage.png
[12]: README_IMAGE/0af52172-eb82-4819-b634-7bbb5608c43fimage.png
[13]: README_IMAGE/bba4fe89-0338-4623-b4dd-534b44dd1985image.png
[14]: README_IMAGE/23e0b30d-0b1f-45ae-b112-16cf95a6f48cimage.png
[15]: README_IMAGE/ed957ee6-d5be-4f51-a14d-7ee62ac5da74image.png
[16]: README_IMAGE/e3f57e56-b93b-4563-aed1-c2a1d0f1b1f8image.png
[17]: README_IMAGE/fef5e68b-49cc-4156-967d-8fbf31198998image.png
[18]: README_IMAGE/f7783253-f923-4373-bc4d-18f587b0bbeeimage.png
[19]: README_IMAGE/caa7c331-d620-4ece-a800-8c7b2f033fb6image.png
[20]: README_IMAGE/3d082ab6-a885-4f85-9004-7774342f4defimage.png














