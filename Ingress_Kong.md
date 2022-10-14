# Kong
https://docs.konghq.com/install/kubernetes/
https://docs.konghq.com/install/kubernetes/?_ga=2.196695532.195122953.1563518403-599058245.1563518403#postgres-backed-kong
https://hub.kubeapps.com/charts/stable/kong
https://docs.konghq.com/hub/


## kong ingress
https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/tutorials/getting-started.md

helm repo add kong https://charts.konghq.com
helm repo update

kubectl create namespace kong
helm upgrade kong-ingress kong/kong --install --values https://bit.ly/2UAv0ZE --set ingressController.installCRDs=false --namespace kong \
  --set serviceMonitor.enabled=true --set proxy.enabled=true --set admin.enabled=true --set manager.enabled=true --set portal.enabled=true

helm upgrade kong-ingress kong/kong --install --set ingressController.installCRDs=false --namespace kong

helm install --name kong-ingress stable/kong --set ingressController.enabled=true --set proxy.type=LoadBalancer --set postgresql.enabled=false --set env.database=off --namespace kong

IP=
DB_HOST=kong.postgres.database.azure.com
DB_PW=
DB_USER=kongadmin
env:
  database: postgres
  pg_host: kong.postgres.database.azure.com
  pg_password: 
  pg_user: 

helm install --name kong-ingress stable/kong --set ingressController.enabled=true --set proxy.type=LoadBalancer --set proxy.loadBalancerIP=$IP --set postgresql.enabled=false --set env.database=kong   --set env.pg_host=$DB_HOST --set env.pg_password=$DB_PW --set env.pg_user=$DB_USER --set env.pg_port=5432 --namespace kong

helm install --name kong-ingress stable/kong --set ingressController.enabled=true --set proxy.type=LoadBalancer --set proxy.loadBalancerIP=$IP --set env.cassandra_contact_points= --set env.cassandra_port=
--namespace kong
	
helm install --name kong-ingress stable/kong --set ingressController.enabled=true --set proxy.type=LoadBalancer --set proxy.loadBalancerIP=$IP --namespace kong

export PROXY_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" service -n kong kong-ingress-kong-proxy)

kubectl apply -f https://raw.githubusercontent.com/denniszielke/container_demos/master/yaml/echoserver.yaml

echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo
  annotations:
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /foo
        backend:
          serviceName: echo
          servicePort: 80
" | kubectl apply -f -
ingress.extensions/demo created


# GRPC

https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/guides/using-ingress-with-grpc.md


KONG_PROXY_IP=20.56.196.127 

grpcurl -v -d '{"greeting": "Kong Hello world!"}' -insecure $KONG_PROXY_IP:443 hello.HelloService.SayHello

cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: httpbin-free
  annotations:
    kubernetes.io/ingress.class: kong
    konghq.com/protocols: http
spec:
  rules:
  - http:
      paths:
      - path: /free
        backend:
          serviceName: httpbin
          servicePort: 80
EOF

cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: httpbin-paid
  annotations:
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /paid
        backend:
          serviceName: httpbin
          servicePort: 80
EOF

KONG_PROXY_IP=20.56.196.127 

curl -X GET $KONG_PROXY_IP/free/status/200 --insecure


## Monitoring

echo 'apiVersion: configuration.konghq.com/v1
kind: KongClusterPlugin
metadata:
  name: prometheus
  annotations:
    kubernetes.io/ingress.class: kong
  labels:
    global: "true"
plugin: prometheus
' | kubectl apply -f -


while true;
do
  curl http://$KONG_PROXY_IP/billing/status/200
  curl http://localhost:8000/billing/status/501
  curl http://localhost:8000/invoice/status/201
  curl http://localhost:8000/invoice/status/404
  curl http://localhost:8000/comments/status/200
  curl http://localhost:8000/comments/status/200
  sleep 0.01
done


## ACL


https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/guides/configure-acl-plugin.md


PROXY_IP=20.56.196.127
curl -i $PROXY_IP/get


echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: app-jwt
plugin: jwt
" | kubectl apply -f -


echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-get
  annotations:
    konghq.com/plugins: app-jwt
    konghq.com/strip-path: "false"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /get
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -


echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-post
  annotations:
    konghq.com/plugins: app-jwt
    konghq.com/strip-path: "false"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /post
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -


curl -i --data "foo=bar" -X POST $PROXY_IP/post


echo "
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: admin
  annotations:
    kubernetes.io/ingress.class: kong
username: admin
" | kubectl apply -f -


echo "
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: plain-user
  annotations:
    kubernetes.io/ingress.class: kong
username: plain-user
" | kubectl apply -f -


echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: plain-user-acl
plugin: acl
config:
  whitelist: ['app-user','app-admin']
" | kubectl apply -f -


echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-post
  annotations:
    konghq.com/plugins: app-jwt,admin-acl
    konghq.com/strip-path: "false"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /post
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -

https://linkerd.io/2/tasks/using-ingress/#kong


## Linkerd

echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: set-l5d-header
  namespace: emojivoto
plugin: request-transformer
config:
  add:
    headers:
    - l5d-dst-override:$(headers.host).svc.cluster.local
" | kubectl apply -f -


echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  namespace: emojivoto
  annotations:
    kubernetes.io/ingress.class: "kong"
    konghq.com/plugins: set-l5d-header
spec:
  rules:
    - host: 51.136.80.117.xip.io
      http:
        paths:
          - path: /api/vote
            backend:
              serviceName: web-svc
              servicePort: http
          - path: /api/vote
            backend:
              serviceName: web-svc
              servicePort: http
          - path: /
            backend:
              serviceName: web-svc
              servicePort: http
" | kubectl apply -f -


kubectl get deployment kong-ingress-kong -n kong -o yaml | linkerd inject --ingress - | kubectl apply -f -
kubectl get deployment <ingress-controller> -n <ingress-namespace> -o yaml | linkerd inject --ingress - | kubectl apply -f -



kubectl edit deployment vote-bot -n emojivoto
env:
# Target the Kong ingress instead of the Emojivoto web service
- name: WEB_HOST
  value: kong-ingress-kong-proxy.kong:80
# Override the host header on requests so that it can be used to set the l5d-dst-override header
- name: HOST_OVERRIDE
  value: web-svc.emojivoto



curl -X POST "https://login.microsoftonline.com/<your_tenant_id>/oauth2/v2.0/token" \
--data scope="<your_client_id>/.default" \
--data grant_type="client_credentials" \
--data client_id="<your_client_id>" \
--data client_secret="<your_client_secret>"


curl --header 'Authorization: bearer <token_from_above>' '<admin-hostname>:8000/httpbin-azure'
