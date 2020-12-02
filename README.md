# Istio manipulation
This is a reference for manipulating the envoy's configuration via Istio-pilot

## Add a ServiceEntry
This would create the virtual host, route, cluster, endpoint accordingly
```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: add-ServiceEntry
  namespace: zkf2
spec:
  hosts:
  - what.is.a.host
  ports:
  - number: 9999
    name: mavService
    protocol: HTTP
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
    - address: 192.160.176.234
```
output:
```
[root@centos8-vm istio-1.7.0]# istioctl pc cluster fortio-bcbf868b9-fngzk.zkf2
SERVICE FQDN                             PORT     SUBSET     DIRECTION     TYPE             DESTINATION RULE
what.is.a.host                           9999     -          outbound      EDS

[root@centos8-vm istio-1.7.0]# istioctl pc endpoints fortio-bcbf868b9-x9ftp.zkf2
ENDPOINT                         STATUS      OUTLIER CHECK     CLUSTER
192.160.176.234:9999             HEALTHY     OK                outbound|9999||what.is.a.host

``` 
## Modify a Cluster
This can change the configuration of an existed cluster, MERGE cannot override old config, need to use REPLACE after istio v1.8
```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: modify-cluster
  namespace: zkf2
  spec:
  configPatches:
  - applyTo: CLUSTER
    match:
      context: SIDECAR_OUTBOUND
      cluster:
        name: "PassthroughCluster"
    patch:
      operation: MERGE
      value:
        original_dst_lb_config:
          use_http_header: true 
```

## Add a Cluster
This can add a new cluster and accordingly endpoint
```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: add-cluster
  namespace: zkf2
spec:
  configPatches:
  - applyTo: CLUSTER
    match:
      context: SIDECAR_OUTBOUND
      cluster:
        name: "outbound|9999||what.is.a.host"
    patch:
      operation: INSERT_AFTER
      value:
        name: "hardcodecluster"
        connect_timeout: "10s"
        typed_config:
          "@type": "type.googleapis.com/envoy.config.cluster.v3.Cluster"
        load_assignment:
          cluster_name: "hardcodecluster"
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: "192.160.176.234"
                    port_value: 9999
```
output:
```
[root@centos8-vm istio-1.7.0]# istioctl pc endpoints fortio-bcbf868b9-x9ftp.zkf2
ENDPOINT                         STATUS      OUTLIER CHECK     CLUSTER
192.160.176.234:9999             HEALTHY     OK                hardcodecluster

[root@centos8-vm istio-1.7.0]# istioctl pc cluster fortio-bcbf868b9-x9ftp.zkf2
SERVICE FQDN                             PORT     SUBSET     DIRECTION     TYPE             DESTINATION RULE        
hardcodecluster                          -        -          -             STATIC
```

## Modify sidecar
In this case, it would 
* Inbound: add route, endpoint, forward the traffic to unix domain socket
* Outbound: add listener to a unix domain socket, attach http_connection_manager to it 
```
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: uds
  namespace: zkf2
spec:
  ingress:
  - port:
      number: 9999
      protocol: HTTP
      name: udsin
    defaultEndpoint: unix:///uds/in.socket
  egress:
  - port:
      number: 0
      protocol: HTTP
      name: udsout
    captureMode: NONE
    bind: unix:///uds/out.socket
    hosts: 
    - "default/*"
    - "./*"
    - "zkf/*"
```

## Replace NetworkFilters
In this case, it would remove the http_connection_manager from listener and attach a tcp_filter to it

```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: replace-filter
  namespace: zkf2
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        name: "unix:///uds/out.socket_0"
        filterChain:
          filter:
            name: "envoy.http_connection_manager"
    patch:
      operation: REMOVE
  - applyTo: NETWORK_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        name: "unix:///uds/out.socket_0"
        filterChain:
    patch:
      operation: ADD
      value:
        name: "envoy.tcp_proxy"
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy"
          stat_prefix: "outbound_unix:///uds/out.socket_0"
          cluster: "PassthroughCluster"
```

## Remove HttpFilters
In this case, it would remove the istio.metadata_exchange from the httpfilter chain
```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: remove-httpfilter
  namespace: zkf2
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        name: "unix:///uds/out.socket_0"
        filterChain:
          filter: 
            name: "envoy.http_connection_manager"
            subFilter:
              name: "istio.metadata_exchange"
    patch:
      operation: REMOVE
```

## Modify HCM Route
In this case, it would change the route in the http_connection_manager
```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: uds
  namespace: zkf2
spec:
  configPatches:
  - applyTo: ROUTE_CONFIGURATION
    match:
      context: SIDECAR_OUTBOUND
      listener:
        name: "unix:///uds/out.socket_0"
        filterChain:
          filter: 
            name: "envoy.http_connection_manager"
    patch:
      operation: MERGE
      value:
        route_config_name: "mav-cluster"
```

## Replace Vhost
In this case, we want to change the route in the route configuration by replacing the vhost in the route 
```
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: replace-vhost
  namespace: zkf2
spec:
  configPatches:
  - applyTo: VIRTUAL_HOST
    match:
      routeConfiguration:
        name: "unix:///uds/out.socket"
        vhost:
          name: "allow_any"
    patch:
      operation: REMOVE
  - applyTo: VIRTUAL_HOST
    match:
      routeConfiguration:
        name: "unix:///uds/out.socket"
        vhost:
          name: "httpbin.default.svc.cluster.local:8000"
    patch:
      operation: INSERT_BEFORE
      value:
        name: "hardcodeFWD"
        domains:
        - "*"
        routes:
        - name: "hardcodeFWD"
          match:
            prefix: "/"
          route:
            cluster: "hardcodecluster"

```

