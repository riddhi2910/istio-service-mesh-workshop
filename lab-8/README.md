# Lab 8 - Mutual TLS & Identity Verification

Istio provides transparent mutal TLS to services inside the service mesh where both the client and the server authenticate each others certificates as part of the TLS handshake. As part of this workshop we have deployed Istio with mTLS.

## 8.1 Verify mTLS
Citadel is Istio’s key management service. Let us first ensure citadel is up and running:
```sh
kubectl get deploy -l istio=citadel -n istio-system
```
Output will be similar to:
```
NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-citadel   1         1         1            1           3h25m
```

To experiment with mTLS, let us get into the sidecar proxy of productpage page by running the command below:
```sh
kubectl exec -it $(kubectl get pod | grep productpage | awk '{ print $1 }') -c istio-proxy -- /bin/bash
```

We are now in the sidecar of the productpage pod. Let us check all the ceritificates loaded in the sidecar:
```sh
ls /etc/certs/
```

You should see 3 entries:
```sh
cert-chain.pem  key.pem  root-cert.pem
```

Let us try to make a curl call to the details service over HTTP
```sh
curl http://details:9080/details/0
```

Since, we have TLS between the sidecar's, an HTTP call will not work. You will see an error like the one below:
```sh
curl: (56) Recv failure: Connection reset by peer
```

Let us try to make a curl call to the details service over HTTPS but **WITHOUT** certs:
```sh
curl https://details:9080/details/0 -k
```

The request will be denied and you will see an error like the one below:
```sh
curl: (35) gnutls_handshake() failed: Handshake failed
```

Now, let us use curl over HTTPS with certificates to the details service:
```sh
curl https://details:9080/details/0 -v --key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k
```

Output will be similar to this:
```sh
*   Trying 10.107.35.26...
* Connected to details (10.107.35.26) port 9080 (#0)
* found 1 certificates in /etc/certs/root-cert.pem
* found 0 certificates in /etc/ssl/certs
* ALPN, offering http/1.1
* SSL connection using TLS1.2 / ECDHE_RSA_AES_128_GCM_SHA256
*        server certificate verification SKIPPED
*        server certificate status verification SKIPPED
* error fetching CN from cert:The requested data were not available.
*        common name:  (does not match 'details')
*        server certificate expiration date OK
*        server certificate activation date OK
*        certificate public key: RSA
*        certificate version: #3
*        subject: O=#1300
*        start date: Thu, 26 Oct 2018 14:36:56 GMT
*        expire date: Wed, 05 Jan 2019 14:36:56 GMT
*        issuer: O=k8s.cluster.local
*        compression: NULL
* ALPN, server accepted to use http/1.1
> GET /details/0 HTTP/1.1
> Host: details:9080
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< content-type: application/json
< server: envoy
< date: Thu, 07 Jun 2018 15:19:46 GMT
< content-length: 178
< x-envoy-upstream-service-time: 1
< x-envoy-decorator-operation: default-route
<
* Connection #0 to host details left intact
{"id":0,"author":"William Shakespeare","year":1595,"type":"paperback","pages":200,"publisher":"PublisherA","language":"English","ISBN-10":"1234567890","ISBN-13":"123-1234567890"}
```

This proves the existence of mTLS between the services on the Istio mesh.

Now lets come out of the container before we go to the next section:

```sh
exit
```


## 8.2 [Secure Production Identity Framework for Everyone (SPIFFE)](https://spiffe.io/)

Istio uses [SPIFFE](https://spiffe.io/) to assert the identify of workloads on the cluster. SPIFFE consists of a notion of identity and a method of proving it. A SPIFFE identity consists of an authority part and a path. The meaning of the path in spiffe land is implementation defined. In k8s it takes the form `/ns/$namespace/sa/$service-account` with the expected meaning. A SPIFFE identify is embedded in a document. This document in principle can take many forms but currently the only defined format is x509.


To start our investigation, let us check if the certs are in place in the productpage sidecar:
```sh
kubectl exec $(kubectl get pod -l app=productpage -o jsonpath={.items..metadata.name}) -c istio-proxy -- ls /etc/certs
```
Output will be similar to:
```sh
cert-chain.pem
key.pem
root-cert.pem
```

MacOS should have openssl available. If your machine does not have openssl install, install it:
```sh
yum install -y openssl
```

Now that we have found the certs, let us verify the certificate of productpage sidecar by running this command:
```sh
kubectl exec $(kubectl get pod -l app=productpage -o jsonpath={.items..metadata.name}) -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout  | grep Validity -A 2
```

Output will be similar to:
```sh
            Not Before: Oct 26 13:25:14 2018 GMT
            Not After : Jan 24 13:25:14 2019 GMT
```

Lets also verify the URI SAN:
```sh
kubectl exec $(kubectl get pod -l app=productpage -o jsonpath={.items..metadata.name}) -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout  | grep 'Subject Alternative Name' -A 1
```

Output will be similar to:
```sh
    X509v3 Subject Alternative Name:
        URI:spiffe://cluster.local/ns/default/sa/default
```
You can see that the subject isn't what you'd normally expect, URI SAN extension has a `spiffe` URI.

This wraps up this lab and the training. Thank you for attending training!

---

# Additional Resources
For future updates and additional resources, check out [layer5.io](https://layer5.io).
