# Governing Knative Eventing With Istio (A Demo)


## First things, first:  _Identity_

### Create wildcard cert 

<!---FIXME should this be using cert manager??? --> 

Initially, we will want to create a CA for our servicemesh installation that wildcards our cluster domain. 

Let's start by creating a root cert to sign a wildcard cert with.  

```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 \
    -subj '/O=Entropic.me/CN=entropic.me' \
    -keyout root.key \
    -out root.crt
```
   
Create a wildcard cert (this will differ by cluster dns, and/or cname usage)

```
openssl req -nodes -newkey rsa:2048 \
    -subj "/CN=*.apps.cluster-3e5f.3e5f.sandbox1022.opentlc.com/O=Enrtopic.me" \
    -keyout wildcard.key \
    -out wildcard.csr

```

Sign the wildcard cert: 

```
openssl x509 -req -days 365 -set_serial 0 \
    -CA root.crt \
    -CAkey root.key \
    -in wildcard.csr \
    -out wildcard.crt

```

And inevitably we'll want to create the wildcard cert as a secret wherever our istio deployment lives (in our case,  _istio-system_ ): 

```
 oc create -n istio-system secret tls wildcard-certs \
    --key=wildcard.key \
    --cert=wildcard.crt
```
