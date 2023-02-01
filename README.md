# Client Certificate Authentication for OpenShift

This uses Request Header authentication and extends OpenShift to support TLS client certificates. Here is the full authentication sequence diagram:

```mermaid
sequenceDiagram
    actor User
    participant console as OpenShift Console
    participant oauth as OpenShift Cluster OAuth
    participant mtls-proxy as Client Certificate Service
    User->>console: start
    console->>oauth: redirect for auth
    oauth->>mtls-proxy: redirect for auth
    User->>mtls-proxy: user athenticates with client certificate
    mtls-proxy-->>oauth: request token
    oauth-->>mtls-proxy: receive token
    mtls-proxy->>console: redirect with token 
    console->>User: logged in
```

## Required Certificates

We need a number of certificates in order to make this work, lets enumerate all
of them:

| What | Why |
| --- | --- |
| Client Certificate CA Chain | Full CA chain of the issuers for the client certificates that will be verified. |
| Frontend | In order to make a TLS connection, the service needs it's own certificates. |
| Backend | Since we will be proxying authentication traffic at minimum we need a certificate pair that is signed and available to the Client Certificate Service AND is trusted by the OpenShift Cluster OAuth |

### Frontend

These certificates should be signed by an issuer that is generally trusted by
browsers because a user will access the service to perform client certificate
authentication.

### Backend

In this example, we will use the [OpenShift Service CA](https://docs.openshift.com/container-platform/4.11/security/certificates/service-serving-certificate.html)
to authenticate the Client Certificate Service to the OpenShift Cluster OAuth service

## Client Certificate Service

We will use Nginx for doing the work of validating the client certificates but
we could also use any number of other solutions including Apache HTTPd.

### Deploy

Clone this git repository and add the client certificate CA as `ca.pem`

```bash
$ git clone <repo>
$ cd <repo>
$ cp ../ca.pem ca.pem
```

Apply the configuration

```
$ oc apply -k .
```

### Configuration

#### Frontend TLS Certificates

By default, the service will deploy using the backend certificates for the frontend.
If we have a new certificate and key we need to first go into the client:

```
$ oc project client-certificate-auth
```

1. Add it as a secret

```
$ oc create secret tls \
    client-certificate-auth-frontend \
    --cert=path/to/server.pem \
    --key=path/to/server-key.pem
```

2. Update the nginx config file `nginx/frontend.conf`:

```diff
server {
...
-    ssl_certificate        /opt/app-root/etc/server-cert/tls.crt;
-    ssl_certificate_key    /opt/app-root/etc/server-cert/tls.key;
+    ssl_certificate        /opt/app-root/etc/frontend-cert/tls.crt;
+    ssl_certificate_key    /opt/app-root/etc/frontend-cert/tls.key;
...
}
```

3. Apply the new configuration

```
$ oc apply -k .
```

#### Fallback Basic Auth with htpasswd

With this authentication module we cannot use other types of authentication in OpenShift
but we can implement any kind of authentication that is available in our service based
on Nginx. In this example, we can use htpasswd-based authenticaton

1. First create a htpasswd file

```
$ htpasswd -c htpasswd jason
```

2. Create a secret from that htpasswd file

```
$ oc create secret generic \
    client-certificate-auth-htpasswd \
    --from-file=htpasswd=path/to/htpasswd
```

3. Roll out the new configuration

```
$ oc rollout restart deployment/auth
```
