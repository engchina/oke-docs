[返回OKE中文文档集](../../README.md)

[返回kubernetes中文文档集](../README.md)

# Kubernetes使用自签名证书

1. 创建自签名证书，示例中使用的域名是`learnk8s.com`，请根据实际情况修改，

   ```
   openssl genrsa -des3 -passout pass:123456 -out ca.key 2048
   openssl rsa -in ca.key -passin pass:123456 -out ca.key
   openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=learnk8s.com"
   openssl genrsa -out tls.key 2048
   openssl req -new -key tls.key -out tls.csr -subj "/CN=*.learnk8s.com"
   cat > server.ext <<EOF
   authorityKeyIdentifier=keyid,issuer
   basicConstraints=CA:FALSE
   keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
   subjectAltName = @alt_names
   
   [alt_names]
   DNS.1 = *.learnk8s.com
   DNS.2 = *.server.learnk8s.com
   EOF
   
   openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 3650 -extfile server.ext
   ```

2. Kubernetes使用自签名证书

   ```
   kubectl create namespace learn-k8s
   kubectl create secret tls server-secret --cert=tls.crt --key=tls.key -n learn-k8s
   ```

3. 验证，

   ```
   cat << EOF | kubectl -n learn-k8s apply -f -
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - image: nginx:1.14.2
           imagePullPolicy: IfNotPresent
           name: nginx
           ports:
           - containerPort: 80
             protocol: TCP
           volumeMounts:
           - mountPath: /usr/share/nginx/html
             name: workdir
         initContainers:
         - command:
           - sh
           - -c
           - echo `hostname` > /usr/share/nginx/html/index.html
           image: busybox:1.28
           imagePullPolicy: IfNotPresent
           name: init-index
           volumeMounts:
           - mountPath: /usr/share/nginx/html
             name: workdir
         volumes:
         - emptyDir: {}
           name: workdir
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
   spec:
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app: nginx
     type: ClusterIP
   ---
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: nginx-ingress
   spec:
     ingressClassName: nginx
     rules:
     - host: nginx.server.learnk8s.com
       http:
         paths:
         - backend:
             service:
               name: nginx-service
               port:
                 number: 80
           path: /
           pathType: ImplementationSpecific
     tls:
     - hosts:
       - nginx.server.learnk8s.com
       secretName: server-secret
   EOF
   ```

   

[返回kubernetes中文文档集](../README.md)

[返回OKE中文文档集](../../README.md)

