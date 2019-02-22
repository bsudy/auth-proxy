# SAML Proxy

## Overview

This repo and Docker image provides a proxy server, configured with SAML authentication.

To start, run like this:
```
docker run -ti -p 80:80 -h auth.example.com -v -e BACKEND=https://api.example.com:8443 -e SCHEMA=http barnabassudy/saml-proxy
```

Ports:
* 80 for http proxying

Environment variables:
* `BACKEND` - to requests are proxied to (_mandatory_)
* `PROXY_HOST` - the hostname the proxy is available - falls back to the host name of the container.
* `SCHEMA` - the schema via the proxy is available (defaults to `https`) - _Note:_ This is not the protocol how the proxy accepts. SSL termination is not a responsibility of this image.
* `DISABLE_SSL_VERIFY` - disabled ssl verification, use if the backend SSL certificate is invalid (defaults to false).
* `REMOTE_USER_EMAIL_SAML_ATTRIBUTE` - the SAML attribute to be sent as `Remote-User-Name header`
* `REMOTE_USER_NAME_SAML_ATTRIBUTE` - the SAML attribute to be sent as `Remote-User-Email`
* `REMOTE_USER_PREFERRED_USERNAME_SAML_ATTRIBUTE` - the SAML attribute to be sent as `Remote-User-Preferred-Username`
* `SAML_MAP_<<sampl_field>>` - this will map the `saml_field` to a request header specified by the property. Eg: `SAML_MAP_EmailAddress=X-WEBAUTH-USER` will map `EmailAddress` SAML field to `X-WEBAUTH-USER` request header.

Volumes:
* `/etc/httpd/conf.d/saml_idp.xml` - SAML IPD metadata (_mandatory_)
* `/etc/httpd/conf.d/saml_sp.key` - SAML SP key (generated if not provided)
* `/etc/httpd/conf.d/saml_sp.cert` - SAML SP certificate (generated if not provided)
* `/etc/httpd/conf.d/saml_sp.xml` - SAML SP metadata (generated if not provided)

# Example Use

## Getting SAML metadata
### Auth0

An example IDP can be created at https://auth0.com/. After creating an account, edit the default app's *Addons > SAML2 Web App* settings as follows:

    * Settings > Application Callback URL*: https://auth.example.com/mellon/postResponse

    * Settings > Settings*:
    ```
    {
        "audience":  "https://auth.example.com",
        ...
    }
    ```

    * Usage > Identity Provider Metadata*: Download the metadata xml and make it available as volume.

## Bitium

TODO

## Configuration

### Docker

```
docker run \
  -p 80:80 \
  -h auth.example.com \
  -v <path>/saml_idp.xml:/etc/httpd/conf.d/saml_idp.xml \
  -e BACKEND=https://api.example.com:8443 \
  -e SCHEMA=http  \
  -ti
  barnabassudy/saml-proxy
```

* `-p 80:80` - port is mapped to localhost:80
* `-h auth.example.com` is equivalent with `-e PROXY_HOST=auth.example.com` - waits for requests for this host and will redirect to this host
* `-v <path>/saml_idp.xml:/etc/httpd/conf.d/saml_idp.xml` - provides the SAML metadata as volume
* `-e BACKEND=https://api.example.com:8443` - the url the requests should be proxied to
* `-e SCHEMA=http`
* `-ti` - make it interactive (eg. being able to stop it with Ctrl+C)
* `barnabassudy/saml-proxy` - the name of the Docker image

### Docker Compose

```
version: "2"
services:
  yourservice:
      ...
  saml-proxy:
      image: "barnabassudy/saml-proxy"
      environment:
          BACKEND: "http://yourservice:port"
      ports:
          - "80:80"
      volumes:
          - "<path>/saml_idp.xml:/etc/httpd/conf.d/saml_idp.xml"
```

### Kubernetes

#### ConfigMap

The configmap stores the SAML configuration. You should download the SAML configuration from you Identity Provider
and put into the config map. You can also use `--from-file` switch on `kubectl create configmap` ([Ref](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/))

```
apiVersion: v1
kind: ConfigMap
metadata:
    name: saml-proxy-config
    namespace: <<yournamespace>>
data:
    saml_idp.xml: <EntityDescriptor>...</EntityDescriptor>

```

#### Deployment

* I mount `saml_idp.xml` from the config as a volume.
* `EmailAddress` SAML field is mapped to `X-Webauth-User` header. In my config the Grafana usernames will be the email addresses from SAML IDP. You can change it to a field you want to use.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: saml-proxy
  namespace: <<yournamespace>>
  labels:
    version: "latest"
spec:
  strategy:
    rollingUpdate:
      maxSurge: 30%
      maxUnavailable: 30%
  progressDeadlineSeconds: 120
  revisionHistoryLimit: 5
  replicas: 1
  selector:
    matchLabels:
      app: saml-proxy
  template:
    metadata:
      labels:
        app: saml-proxy
        version: "latest"
    spec:
      containers:
      - name: saml-proxy
        image: "barnabassudy/saml-proxy:latest"
        ports:
        - name: http-service
          containerPort: 80
        livenessProbe:
           tcpSocket:
             port: http-service
           initialDelaySeconds: 10
        # Mounting saml_idp.xml from configMap
        volumeMounts:
        - name: saml-volume
          mountPath: /etc/httpd/conf.d/saml_idp.xml
          subPath: saml_idp.xml
        env:
        - name: BACKEND
          value: "http://<<yourservice>>"
        - name: REMOTE_USER_SAML_ATTRIBUTE
          value: login
        # You can specify multiple mappings
        - name: SAML_MAP_EmailAddress
          value: X-WEBAUTH-USER
      volumes:
        - name: saml-volume
          configMap:
            name: saml-proxy-config
```

#### Service

To expose saml-proxy as a service

```
apiVersion: v1
kind: Service
metadata:
  name: saml-proxy
  namespace: <<yournamespace>>
spec:
  ports:
    - name: http
      port: 80
      targetPort: http-service
  selector:
    app: saml-proxy
```

#### Ingress

To make it available in the Ingress controller.

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: monitor
  namespace: <<yournamespace>>
  annotations:
    # USE YOUR ANNOTATIONS
    # To enable let's encrypt with kube-logo: kubernetes.io/tls-acme: "true"
spec:
  # Use some SSL
  # tls:
  # - hosts:
  # # The host where the service will be available
  #   - "grafana.example.com"
  #   secretName: grafana-tls
  rules:
  # The host where the service will be available
  - host: "grafana.example.com"
    http:
      paths:
      - path: /
        backend:
          serviceName: saml-proxy
          servicePort: http
```
