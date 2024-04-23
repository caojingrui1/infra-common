# 说明
在CCI中我们使用Service加多个Deployment的模式，为集群内部的环境提供本地Cache，具体组件如下，

## 负载均衡Service
service可以为多个peer pod提供负载均衡功能，CCI中的Space Pod通过该Service利用其中某个pod做代理，避免单个Pod的网络和存储带宽上限, 我们这里创建的CCI内部用于负载均衡的Servic为:
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    email: tommylikehu@gmail.com
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"email":"tommylikehu@gmail.com"},"labels":{"owner":"tommylike"},"name":"dragonfly-cache-service","namespace":"dragonfly-cache"},"spec":{"ports":[{"name":"proxy-service","port":65001,"protocol":"TCP","targetPort":65001}],"selector":{"component":"dragonfly-cache","owner":"tommylike"},"type":"ClusterIP"}}
  creationTimestamp: "2024-02-23T08:44:21Z"
  labels:
    owner: tommylike
  name: dragonfly-cache-service
  namespace: dragonfly-cache
  resourceVersion: "539181262"
  selfLink: /api/v1/namespaces/dragonfly-cache/services/dragonfly-cache-service
  uid: 97fae54e-fb0a-4810-a052-be4bc967fffe
spec:
  clusterIP: 10.247.21.13
  ports:
  - name: proxy-service
    port: 65001
    protocol: TCP
    targetPort: 65001
  selector:
    component: dragonfly-cache
    owner: tommylike
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```
## Peer单实例绑定的Service
这个Service跟负载均衡的Service不同，他绑定了ELB，核心是为了把Peer Pod能暴露到整个VPC中，这样，其他非CCI节点也可以跟这个Peer建立连接，能最大程度复用我们的Peer节点，他们对应的关系是
```
ELB                          ->                                 Service             ->              Deployment         ->      Pod
cdb6ee07-bea9-4afa-8575-b4a73a122399 172.21.0.103  ->  dragonfly-cache-peer1-service  -> dragonfly-cache-deployment-1  -> deployment1-Pod  
ed2b7710-3772-4bbb-9c35-ddb4f9af271c 172.21.0.11  ->  dragonfly-cache-peer2-service  -> dragonfly-cache-deployment-2  -> deployment2-Pod  
60f2dbfe-584c-4d33-922e-69551a723bec 172.21.0.82  ->  dragonfly-cache-peer3-service  -> dragonfly-cache-deployment-3  -> deployment3-Pod  
949775f9-de88-448d-af8e-c0cd8432ae36 172.21.0.33  ->  dragonfly-cache-peer4-service  -> dragonfly-cache-deployment-4  -> deployment4-Pod  
3135a597-1769-4c20-ab0f-11b366dece01 172.21.0.49   ->  dragonfly-cache-peer5-service  -> dragonfly-cache-deployment-5  -> deployment5-Pod  

```
## Deployment
实际会部署多个Deployment，而不是部署1个Deployment的多个副本，核心是使用独立的磁盘，避免同时读写磁盘造成的读写速度上限，当前peer也不支持同时读写一个磁盘。

## Init Container
Peer的环境，因为配置文件涉及配置Pod的IP地址，需要补充init container 用于IP地址的修改
