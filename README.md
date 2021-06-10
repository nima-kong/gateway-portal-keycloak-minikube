# Installing Konnect API Gateway and securing it using KeyCloak

A step by step series of examples that tell you how to set up an enterprise gateway and KeyCload in development environment.
We integrate API Gateway and Dev Portal with KeyCloak.

## Start Minikube
Set default settings for Minikube

```
minikube config set memory 7168
minikube config set cpus 4
minikube config set driver docker
```

Start Minikube

```
minikube start
```

## Install KeyCloak

Create iam namespace
```
kubectl create namespace iam
```

Install KeyCloak server.Default username and password is admin/admin

```
kubectl create -f ./keycloak.yaml -n iam
```
Start a tunnel in Minikube and leave it running

```
sudo minikube tunnel
```

Edit your hosts file and add the following entry.
```
127.0.0.1     keycloak.iam.svc.cluster.local
```
Login to KeyCloak (http://keycloak.iam.svc.cluster.local:8080/auth/) and add the kong client.
```
Please go to configure-> clients
Click on Create
Enter kong in ClientID
Click Save.
Make sure Access Type is confidential and Service Accounts is enabled
```
Add the following redirect URLs to the Valid Redirect URLs in kong client and click save.
```
http://kong.local:8002/*
http://kong.local:8003/*
http://kong.local:8004/*
```
Take note of secret for the kong client. You can find it in the Credentials tab.

Add a new group

```
Go to Manage->Groups
Add a new group
Click New
Enter super-admin
Click save
```
Create kong admin user
```
Go to Manage->Users
Click Add user
Set Username to kong_admin
Click save
Select kong_admin user
Go to credentials tab and set the password, disable Temporary
Add the user to super-admin group
```
Create Dev Portal user
```
Go to Manage->Users
Click Add user
Set Username to test@myco.com
Click save
Select test@myco.com user
Go to credentials tab and set the password, disable Temporary
```

## Install Konnect Gateway

Create kong namespace
```
kubectl create namespace kong
```
Create a secret to hold the license information
```
kubectl create secret generic kong-enterprise-license --from-file=license=license.json -n kong
```

Create a password for Kong kong_admin user. Please replace <Password>
```
kubectl create secret generic kong-enterprise-superuser-password -n kong --from-literal=password=<Password>
```

Please make sure you replace the <Client_Secret> with the one you have.
Install Konnect Gateway
```
kubectl apply -f kong-ee-with-ingress-oidc.yaml
```
Make sure all the pods are running
```
kubectl get pods -n kong
```
Edit your hosts file and add the following entry.
```
127.0.0.1       kong.local
```

## Configure Dev Portal

**Note:** In order to avoid conflict between different user's session, it is recommended to open incognito window and close it after completing the instruction.

Please log out of KeyCloak and close your browser. 
Open a new browser page in incognito and set up Dev Portal
```
Go to http://kong.local:8002/default/dashboard
Login as kong_admin user
Go to Default workspace
Go to Dev Portal-> Overview and enable the Dev Portal
Select Dev Portal->Settings
Make sure Authentication plugin is set to Open ID Connect.
And Auto Approve Access is enabled
```
Use Custom option in Auth Config(JSON) and past the following. Please make sure you replace the <Client_Secret> with the one you have.
```
{
    "leeway": 1000,
    "consumer_by": [
        "username",
        "custom_id",
        "id"
    ],
    "scopes": [
        "openid",
        "profile",
        "email",
        "offline_access"
    ],
    "logout_query_arg": "logout",
    "consumer_claim": [
        "preferred_username"
    ],
    "login_action": "redirect",
    "logout_redirect_uri": [
        "http://kong.local:8003"
    ],
    "logout_methods": [
        "GET"
    ],
    "login_redirect_uri": [
        "http://kong.local:8003"
    ],
    "forbidden_redirect_uri": [
        "http://kong.local:8003/unauthorized"
    ],
    "issuer": "http://keycloak.iam.svc.cluster.local:8080/auth/realms/master",
    "client_id": [
        "kong"
    ],
    "ssl_verify": false,
    "client_secret": [
        "<Client_Secret>"
    ],
    "login_redirect_mode": "query"
}
```
Log out of kong manager and create a new user in Dev Portal
```
Go to http://kong.local:8003/default
Create a Developer Account
Full Name: Test
Email: test@myco.com

```
Now you can login to the Dev Portal. Please use an incognito window to avoid conflicting with any existing sessions.

## Enable Application Registration

In this section, we will create an application in KyCloak and use it to call an API in Kong. This API is published to Dev portal and secured using OIDC plugin

First we need to create a client in Keycloak

```
Go to clients tab in Keycloak
Create a new client and call it testapp
Set Access Type to confidential
Enable service account
Add http://kong.local:8000/* as a valid redirect UI
Click Save

```

Now, we are going to create an API called mockbin and apply application-registration and openid-connect plugins to it

```
curl -i -X POST http://localhost:8001/default/services/ \
   --header 'kong-admin-token: default_admin' \
   --data "name=mockbin" \
   --data "url=http://mockbin.org/request"
```

```
 curl -i -X POST http://localhost:8001/default/services/mockbin/routes \
   --header 'kong-admin-token: default_admin' \
   --data "name=mockbin" \
   --data "paths[]=/mockbin"
```

```
http -f post :8001/routes/mockbin/plugins \
   name=openid-connect \
   config.issuer=http://keycloak.iam.svc.cluster.local:8080/auth/realms/master \
   config.verify_parameters=false \
   config.consumer_claim=clientId \
   config.ssl_verify=false \
   kong-admin-token:default_admin
```

```
http -f post :8001/services/mockbin/plugins \
   name=application-registration \
   config.display_name=mockbin \
   kong-admin-token:default_admin
```

In the next step, we create an aplication in Dev Portal

```
Go to http://kong.local:8003/default
Login as test@myco.com
Go to My Apps tab
Create new Application
 - Name=testapp, Reference Id=testapp

```
Lets activate the mockbin service fo testapp
```
From list of applications select View for testapp
In the Services section of the page, click Active infront of mockbin service
In Kong Gateway, go to http://kong.local:8002/default/applications
Select View infront of testapp
In the Service Contracts section of the page, select Requested Access tab and click approve for testapp request

```
Use the following request to call your API. Remember to replace <client_secret> with testapp client secret
```
http :8000/mockbin -a testapp:<client_secret>
```



