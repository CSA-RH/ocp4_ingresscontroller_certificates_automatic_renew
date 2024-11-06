# Introduction

This repository contains an example of managing the wildcard certificate of a custom IngressController using cert-manager.

The Ingress Controller employs a default certificate for secure routes unless a custom certificate is explicitly specified. Here, the objective is to utilize cert-manager to generate a custom certificate issued by a self-signed certificate authority (CA). Subsequently, we will update the ingress controller's default certificate to use this newly generated certificate.

All the configurations below are only meant for demonstration purposes and must be tweaked to address the particular use case requirements.

# Procedure
### 1.Define a Private Domain that will be configured in the Custom IngressController and to which the Certificates will be served.
In my case Im using the domain: "mydomain.com"

### 2.Install the cert-manager operator
e.g. chapter "Installing the cert-manager Operator" of:
https://docs.openshift.com/rosa/cloud_experts_tutorials/cloud-experts-dynamic-certificate-custom-domain.html

### 3.Create customized IngressController
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: custom-domain-ingress
  namespace: openshift-ingress-operator
spec:
  domain: ${DOMAIN}
  defaultCertificate:
    name: custom-domain-ingress-cert-tls
  endpointPublishingStrategy:
    loadBalancer:
      dnsManagementPolicy: Unmanaged
      providerParameters:
        aws:
          type: NLB
        type: AWS
      scope: Internal      #Internal means it is private, i.e. apps are not exposed to internet.
    type: LoadBalancerService


### 4.Create a self-signed CA issuer
This step will create a self-signed CA certificate, this CA certificate will be used to sign the Certificate requests.
https://cert-manager.io/docs/configuration/selfsigned/

### 4.1.
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer #----(1)
spec:
  selfSigned: {}

### 4.2.
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-root-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: selfsigned-root-ca # Note this
  secretName: root-ca-key-pair #----(2)
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-cluster-issuer # match (1)
    kind: ClusterIssuer
    group: cert-manager.io

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: root-ca-key-pair  #----(2)

### 4.3.To verify the ClusterIssuer and Certificate were created correctly
oc get ClusterIssuer ca-issuer -o yaml
oc get ClusterIssuer  selfsigned-cluster-issuer -o yaml
oc get Certificate -n cert-manager -o yaml


### 5.Issue the custom IngressController custom Wildcard Certificate

### 5.1.
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ingress-wildcard-cert
  namespace: openshift-ingress
spec:
  isCA: false
  commonName: "*.${DOMAIN}" # change me
  dnsNames:
    - "${DOMAIN}"           # change me
    - "*.${DOMAIN}"         # change me
  usages:
    - server auth
  issuerRef:
    kind: ClusterIssuer
    name: ca-issuer
  secretName: ingress-wildcard-tls

### 5.2.Check that certificate was created
oc -n openshift-ingress describe Certificate ingress-wildcard-cert 

### 6.Change the custom IngressController default certificate
### 6.1.
oc patch ingresscontroller.operator custom-domain-ingress \
--type=merge -p \
'{"spec":{"defaultCertificate": {"name": "ingress-wildcard-tls"}}}' \
-n openshift-ingress-operator

This will restart the custom Router pods in the openshift-ingress namespace.

### 6.2.Verify the update was effective:
echo Q | openssl s_client -connect console-openshift-console.${DOMAIN}:443 -showcerts 2>/dev/null | openssl x509 -noout -subject -issuer -enddate

# Testing
The client where the curl was generated is my Laptop. I have established a VPN from my Laptop towards AWS and, for a fast test, added an entry in /etc/hosts containing: hello2.${DOMAIN} pointing to one private IP of the AWS NLB deployed by the custom IC. 
oc new-project hello-world2
oc -n hello-world2 new-app --image=docker.io/openshift/hello-openshift
oc -n hello-world2 create route edge --service=hello-openshift hello-openshift-tls --hostname hello2.${DOMAIN}
curl -I https://hello2.${DOMAIN} -k
     
# References
https://docs.openshift.com/container-platform/4.17/security/cert_manager_operator/cert-manager-creating-certificate.html#cert-manager-certificate-ingress_cert-manager-creating-certificate
https://docs.openshift.com/rosa/cloud_experts_tutorials/cloud-experts-dynamic-certificate-custom-domain.html
https://developers.redhat.com/learning/learn:openshift:simplify-certificate-management-openshift-across-multiple-architectures/resource/resources:automate-tls-certificate-management-using-cert-manager-operator-openshift?source=sso
NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.
