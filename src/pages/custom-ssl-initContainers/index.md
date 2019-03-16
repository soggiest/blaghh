---
title: Custom SSL Certs in Kubernetes using initContainers
date: "2019-03-14"
---
# Trusting custom SSL certs in Kubernetes using initContainers

One of the issues I've run into many times while deploying applications in Kubernetes is how to get a custom or self-signed cert to be trusted by the application without building a new docker image. You may run into this situation if you're deploying workloads from a restricted repository, or you don't want to modify the base image for a container. Luckily there's a useful mechanism you can use to get around this: initContainers.

Check out this example which trusts a custom SSL cert used by the main container running Gangway (https://github.com/heptiolabs/gangway)

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: gangway
  namespace: gangway
  labels:
    app: gangway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gangway
  strategy:
  template:
    metadata:
      labels:
        app: gangway
        revision: "1"
    spec:
      nodeSelector:
        app: gangway
      initContainers:
        - name: gangway-certs
          image: alpine:latest
          imagePullPolicy: Always
          command:
          - sh
          - -c
          - apk add --update ca-certificates openssl && cp /var/tmp/* /usr/local/share/ca-certificates && update-ca-certificates && cp /etc/ssl/certs/* /tmp/
          volumeMounts:
          - mountPath: /var/tmp
            name: custom-cert
          - mountPath: /tmp/
            name: ssl-certs
      containers:
        - name: gangway
          image: gcr.io/heptio-images/gangway:v3.0.0
          imagePullPolicy: Always
          command: ["gangway", "-config", "/etc/gangway/cfg/config.yaml"]
          env:
            - name: GANGWAY_SESSION_SECURITY_KEY
              valueFrom:
                secretKeyRef:
                  name: gangway-key
                  key: sesssionkey
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          volumeMounts:
            - name: custom-cert
              mountPath: /etc/gangway
            - mountPath: /etc/ssl/certs
              name: ssl-certs
      volumes:
        - secret:
          name: ca-tls
          items:
          - key: tls.crt
            path: tls/ca.pem
          - key: tls.key
            path: tls/ca.key
        - configMap:
          name: gangway
          items:
          - key: config.yaml
            path: cfg/config.yaml
        - name: ssl-certs
          emptyDir: {}
```
NOTE: This deployment requires a few things to be present before it works properly, namely the cert Secrets, Gangway's configmap, and the Gangway session key secret. So, don't expect to copy and paste this and it'll just work.

Let's take a look at what's going:

* The gangway-certs initContainer mounts the `custom-cert` secret, which should contain your custom certs, and the `ssl-certs` emptyDir volume, which instructs the container to create a new directory at the location of the mountPath.
* The initContainer then runs through the steps specified in the `commands` section, which:
  * Adds openssl and ca-certificates binaries since they don't exist on `alpine` images.
  * Copies the custom certs to `/usr/local/share/ca-certificates`, which is the default location for ca-certificates to look for new certs.
  * Runs `update-ca-certificates`, which adds the new certs to `/etc/ssl/certs` in the appropriate format.
  * Copies the `/etc/ssl/certs/` to the `ssl-certs` emptyDir volume

The initContainer and the main container share the `ssl-certs` directory, which now contains a list of trusted certs. The main `Gangway` container mounts the `ssl-certs` volume to `/etc/ssl/certs` directory, which is the root directory for trusted certs. With your custom cert now in this directory when the container starts up it will be trusted by Gangway when it starts. All without changing the base image. Neat, huh?

---
