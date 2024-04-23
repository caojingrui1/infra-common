# 说明
在CCE环境中，我们通过部署deamonset，来提供本地加速环境

## 负载均衡Service
Service可以为多个peer pod提供负载均衡功能，CCE中的Space Pod通过该Service(internalTrafficPolicy：Local)利用跟其在同个节点的Pod做加速缓存。