# Helm Chart for FDP Pelican-Cache

This Helm Chart is a work in progress to deploy a
Pelican-cache to OpenShift-managed clusters at sites
wishing to access Fusion Data Platform (FDP) data.
Specifically, it uses the `/fdp-d3d` namespace within
<https://osg-htc.org>.



# Installation Instructions

This guide gives instructions to install it into an OpenShift
cluster using [oc](https://github.com/openshift/oc) and [helm](https://helm.sh/docs/intro/install/).

## Step 1 - login to OpenShift

    oc login
    oc project <projectname>

## Step 2 - list available storage classes

You need to know the types of storage classes
available before you can create a
persistent volume claim (PVC).

    oc get sc

Ideally, there's one that's NFS-backed in there.
If there isn't one, check with your Kubernetes cluster
admin on allocating disks for this purpose.

## Step 3 - create the pelican-lib volume

```
oc apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pelican-lib
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 2Gi
  storageClassName: <YOUR_NFS_STORAGECLASS>
EOF
```

## Step 4 - deploy the (container, service, route) bundle

    helm install pelican-cache ./pelican-cache

## Step 5 - fix the internal certificate

    oc -n <ns> get configmap service-ca-bundle \
      -o jsonpath='{.data.service-ca\.crt}' > service-ca.crt

    oc patch route pelican-cache-web-ui \
      --type=merge \
      -p "{\"spec\":{\"tls\":{\"termination\":\"reencrypt\",\"destinationCACertificate\":\"$(awk '{printf \"%s\\n\", $0}' service-ca.pem)\"}}}"

This is needed so the reencrypt TLS termination will pass traffic
from the OpenShift router to the pelican-cache pod.  Otherwise it
would not be able to verify the pod's x509 certificate.


# TLS Certificates

A large part of the configuration is using x509 certificates
correctly.

## Option A: you supply a secret

Create a TLS secret on your own, then upload it to OpenShift.

    oc create secret tls pelican-webui-tls --cert=cert.pem --key=key.pem

Mount it into /etc/pelican and point pelican at it.

Then set the Route’s destinationCACertificate to the CA that issued cert.pem (or use a cert issued by the OpenShift service-ca and trust that).


## Option B: OpenShift-native

OpenShift can automatically create a serving cert/key Secret for a Service by annotation. The pod can mount that Secret and use it as its TLS identity.  This has been setup in the current config.

Typical flow:

* Annotate the Service with service.beta.openshift.io/serving-cert-secret-name: <secretName>
* OpenShift creates/rotates a Secret <secretName> containing tls.crt and tls.key
* Mount that Secret into the pod and configure pelican to use it
* Put the service-ca bundle into the Route’s destinationCACertificate (so router trusts the backend)

This gives you an OpenShift-issued keypair inside the pod, but it is not the router’s keypair; it’s a backend serving cert for your Service.

If using this option, you will need to paste your Kubernetes
cluster's CA-bundle into the `pelican-cache/route.yaml` config
file.  However, you usually can't get that information until
after you've deployed the service.

    oc -n <namespace> get configmap service-ca-bundle \
      -o jsonpath='{.data.service-ca\.crt}' > service-ca.crt

# References

* [github:bbockelm/pelican-cache](https://github.com/bbockelm/pelican-cache) - Production Pelican Cache Helm Chart Reference

  Uses certmanager and letsencrypt, rather than relying on the Kubernetes route to have its own certificate.



