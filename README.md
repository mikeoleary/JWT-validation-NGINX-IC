# JWT-validation-NGINX-IC
Files for deploying demo application with:
1. JWT signature verification
2. arbitrary claims validation
3. forwarding claims values as headers 

# Prerequisites
- NGINX Plus requires a license. Build your NGINX Ingress Controller using your license, or pull the image from a private registry with your credentials.
  - The file ````docker-login-secret.yaml```` should be updated to reflect your private registry and credentials.
- This demo doesn't cover exposing the NGINX service to outside of the cluster. It assumes F5 CIS for this purpose, and the file ````nginx-ingress/vs-ts.yaml```` is used for this purpose.
- This demo was built for an Openshift use case. For other Kubernetes distributions, minor changes may be required.
- **This demo is intended to use insecure secrets. Do not use these secrets in production.** 
  - I have used __nginx123__ as the secret for JWT signature verification.
  - I have included 2x sample JWT tokens. The files in directory ````sample-JWT-tokens```` are simply for educational purposes. They use the secret __nginx123__
  - The NGINX default server secret is an example, taken from NGINX's website.

# Instructions for demo

Clone this repo and cd into the directory to run the commands below.
- you will need to update the annotations in the file ````nginx-ingress/common/ns-and-sa.yaml```` with the IP address of your internal SelfIP. This is only for Openshift users, where OVN-Kubernetes hybrid overlay feature is used.
- you will need to update the file ````nginx-ingress/vs-ts.yaml```` if you are using CIS to expose the NGINX I.C. service.
- you will need to update ````nginx-ingress/docker-login-secret.yaml```` with your credentials and address of your private registry.

````bash
#deploy Demo app
kubectl apply -f demoapp/namespace.yaml
kubectl apply -f demoapp/f5demoapp.yaml

#deploy NGINX Plus as Ingress Controller. These commands assume the pull secret is already created.
kubectl apply -f nginx-ingress/common/ns-and-sa.yaml
kubectl apply -f nginx-ingress/rbac/rbac.yaml
kubectl apply -f nginx-ingress/common/default-server-secret.yaml
kubectl apply -f nginx-ingress/common/docker-login-secret.yaml
kubectl apply -f nginx-ingress/common/ingress-class.yaml
kubectl apply -f nginx-ingress/common/nginx-plus-config.yaml
kubectl apply -f nginx-ingress/common/crd/k8s.nginx.org_policies.yaml
kubectl apply -f nginx-ingress/common/crd/k8s.nginx.org_transportservers.yaml
kubectl apply -f nginx-ingress/common/crd/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f nginx-ingress/common/crd/k8s.nginx.org_virtualservers.yaml
oc adm policy add-scc-to-user privileged system:serviceaccount:nginx-plus-ingress:nginx-plus-ingress # this command is required in Openshift environment to allow NGINX Plus I.C. to bind to port 80
kubectl apply -f nginx-ingress/deployment/nginx-plus-ingress.yaml
kubectl apply -f nginx-ingress/service/service.yaml
kubectl apply -f nginx-ingress/vs-ts.yaml
````

__Option 1__: To configure multiple Ingress resources to route traffic via NGINX Ingress Controller, run the below commands:

````bash
#create jwk secret, and use Ingresses to configure JWT signature verification and claims validation
kubectl apply -f nginx-ingress/jwk-secret.yaml
kubectl apply -f demoapp/ingress-demoapp-master.yaml
kubectl apply -f demoapp/ingress-demoapp-base.yaml
kubectl apply -f demoapp/ingress-demoapp-headers.yaml
````

__Option 2__: To configure VirtualServer, Policy, and VirtualServer route resources to route traffic via NGINX Ingress Controller, run the below commands:

````bash
#create jwk secret, and use Ingresses to configure JWT signature verification and claims validation
kubectl apply -f nginx-ingress/jwk-secret.yaml
kubectl apply -f demoapp/policy.yaml
kubectl apply -f demoapp/virtualserver.yaml
kubectl apply -f demoapp/virtualserverroute.yaml
````

# Validate your configuration

Now, access your site. This demo uses __demo.my-f5.com__ as the url for the demo web application. Check the following are true. Use a private browser and restart the browser for every step of validation, to avoid confusing sessions and cookies.

1. You can access __demo.my-f5.com__ and hit the page at the site root.
2. You should get a 401 unauthorized error when you try to access /headers.
3. Close the browser, re-open, and browse to __demo.my-f5.com__ again. This time, use dev tools to send a cookie where the name is "jwt" and the value is the base64-encoded JWT token belonging to **"mike"**
4. You should be able to see the home page, as well as the /headers page (i.e. JWT authentication is working). The /headers page should show you a request header named X-jwt-claim-name (ie. header insertion based on arbitrary jwt claim is working)
5. Close the browser, re-open, and browse to __demo.my-f5.com__ again. This time, use dev tools to send a cookie where the name is "jwt" and the value is the base64-encoded JWT token belonging to **"anna"**
6. You should be able to see the home page, but NOT the /headers page (i.e. Authorization using JWT claims validation is working because Anna cannot see the page that Mike can see).
