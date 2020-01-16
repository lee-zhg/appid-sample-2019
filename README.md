# AppID in your Kube Applications with a Custom Login Button

## Introduction

Greetings!

I've been playing around with the AppID IKS ingress annotation, and was very happy to see it work as advertised, without any code changes on my part.

One thing it doesn't support, though, is the ability to have my own application landing page with my own login button.  To do that, I need to use the APIs which can be a tad unwieldy.  Fortunately, I figured out a way to get a login button and not have to do much in the application code to support it. 

This repository includes a working sample you can use as a base to incorporate into your own applications.

## Step 1 - Provision the AppID Service and Add Some Users

Find the AppID service in the IBM Cloud catalogue and provision into your account.  Once provisioned, initiate the service and enable the cloud directory:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid1.png)

Now add a user or two:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid2.png)

## Step 2 - Pull the AppID Service Secret into your namespace:

This step is not strictly neccessary to secure the application. However, this demo does make a call to the AppID service. We therefore need to pull the credentials for AppID into our Kube cluster:

```
ibmcloud ks cluster-service-bind --cluster <ClusterName> --namespace default --service <AppIDServiceName>
```
Replace `<ClusterName>` with the name of the Kube cluster where the demo will run and replace `<AppIDServiceName>` with the name of your AppID service.

Now you can check:

```
[root@jumpserver]# kubectl get secrets | grep binding
binding-appid                  Opaque                                1      6d21h
[root@jumpserver]#
```
In your environment, the secret name will be `binding-<appidservicename>`
  
## Step 3 - Build your Container Image

After pulling this repository down into your environment, build the image directly into your IBM Cloud image repository:

```
 ibmcloud cr build -t us.icr.io/<namespace>/appidsample:1 .
```
Where `<namespace>` is one of your container registry namespaces (you many also need to update the registry location, depending on which region your registry is in).

## Step 4 - Create certificates and secrets

AppID only works on web applications locked down with SSL, so you will need to create certificates and a secret.  This repository contains two scripts, `cert.sh` and `secret.sh` (in that order) to help you with that.  

When you create your certificates, make sure you use the fully qualified hostname that includes your ingress subdomain, as per the following example:

```
[root@jumpserver appid-sample]# ./cert.sh
Generating a 2048 bit RSA private key
................................................+++
.....................+++
writing new private key to 'appid-key.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CA
State or Province Name (full name) []:ON
Locality Name (eg, city) [Default City]:TO
Organization Name (eg, company) [Default Company Ltd]:IBM
Organizational Unit Name (eg, section) []:Cloud
Common Name (eg, your name or your server's hostname) []:appidsample.clustername.us-east.containers.mybluemix.net 
Email Address []:
[root@jumpserver appid-sample]#
```

You can find your ingress subdomain on your cluster overview page in IBM Cloud:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid3.png)

## Step 5 - Deploy the application

In the `yaml` subdirectory of this repository there is a deployment.yaml file.  Before you deploy the application with it, you will need to make two changes.

On line 20, change the image namespace from `us.icr.io/<namespace>/appidsample:1` to whatever namespace you built the image to.

On line 27, change `binding-appid` to the AppID secret name for you environment.

After you make these changes, deploy the application by typing `kubectl create -f ./deployment.yaml`

Check to make sure the pod is up and running:

```
[root@jumpserver appid-sample]# kubectl get pods | grep appid
appidsample-6c5c4f4d6f-vt42g                1/1     Running   0          32h
[root@jumpserver appid-sample]#
```

## Step 6 - Deploy the service

In the `yaml` subdirectory, create the service with `kubectl create -f ./service.yaml`.  No changes to the yaml file is required.

When you check the service, you will actually see two services:

```
[root@jumpserver appid-sample]# kubectl get services | grep appid
appidsample-insecure            ClusterIP      172.21.39.185    <none>           3000/TCP                        41h
appidsample-secure              ClusterIP      172.21.183.48    <none>           3000/TCP                        41h
[root@jumpserver appid-sample]#
```

## Step 7 - Deploy the ingress

Let's have a look at the `ingress.yaml` file in the `yaml` subdirectory:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: appidsample
  annotations:
    ingress.bluemix.net/appid-auth: bindSecret=binding-appid namespace=default requestType=web serviceName=appidsample-secure
    ingress.bluemix.net/redirect-to-https: "True"
spec:
  rules:
  - host: appidsample.clustername.us-east.containers.mybluemix.net
    http:
      paths:
      - backend:
          serviceName: appidsample-secure
          servicePort: 3000
        path: /secure
      - backend:
          serviceName: appidsample-insecure
          servicePort: 3000
        path: /
  tls:
  - hosts:
    - appidsample.clustername.us-east.containers.mybluemix.net
    secretName: appidsampl
```

The AppID ingress annotation will intercept all traffic heading to the `appidsample-secure` service, defined as path `/secure`.  All other traffic will be serviced by the `appidsample-insecure` service.  That's why we defined two services.

Before deploying, change lines 10 and 23 to your fully qualified domain name, which must be the same name you used to create the certificate during step 4

After updating the file, deploy the ingress with `kubectl create -f ./ingress.yaml`

Checking your ingress, you should see something like the following:

```
[root@jumpserver appid-sample]# kubectl get ingress | grep appid
appidsample        appidsample.clustername.us-east.containers.mybluemix.net    169.48.5.14   80, 443   41h
[root@jumpserver appid-sample]#
```

## Step 8 - Add the Callback URL to AppID

You need to register your application with your AppID instance.  You do this by adding a unique IKS URL to your AppID config.

Go back to your AppID instance and select Manage Authentication-->Authentication Settings:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid6.png)


Add the following web direct URL to the configuration, substituting in your fully qualfied hostname:

```
https://appidsample.clustername.us-east.containers.mybluemix.nea/secure/appid_callback 
```

## Step 9 - Test the application

At this point you should be able to surf to the application using the ingress host name:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid4.png)

When you press the `Login` button, you should be redirected to the AppID login page.  After providing your credentials, you should be redirected back to the application, with your user information from AppID dumped to the screen:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid5.png)

Pressing the `Logout` button will log you out of the service.


## Notes

If you take a look at the server code, you will see that all secure operations hang off the `/secure` URL:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid7.png)

Everything hanging off `/secure` will be intercepted by AppID and checked to see if you are logged in and have access to the service.  The resulting call on the server side will contain an `Authorization` header in the request bearing the identity token of the logged in user.  The above sample code uses that token to retreive the client details from AppID.

To log out, IKS also supports an AppID logout URL. However, it always forces a redirect back to the AppID login page.  To avoid this, I added a server side logout function to remove the AppID cookies from the browser:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid8.png)

On the client side, the Javascript to perform logout looks like this:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid9.png)

Note the code comments.  If you have just enabled cloud directory, then you are fine.  However, if you have enabled a SAML provider and want to ensure your credentials are cleared there as well, you need to redirect the browser to the supplied `logout.html`.

This file uses a hidden frame to call the SAML provider secretly to remove your credentials:

![alt text](https://github.ibm.com/robobob/appid-sample/blob/master/images/appid10.png)

You will need to change the URL on line 52 to the correct URL for your SAML provider.  The IBM SAML providers can be found here:

https://appid-iks-federated-logout.antona.us-south.containers.appdomain.cloud/test

Good hunting!


