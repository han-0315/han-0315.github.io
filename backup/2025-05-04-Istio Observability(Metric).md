---
layout: post
title: istio Observability(Metric)
date: 2025-05-03 09:00 +0900 
description: istio Observability(Metric) 살펴보기
category: [Kubernetes, Network] 
tags: [istio, CloudNet, Kubernetes, Network, istio#4, Observability] 
pin: false
math: true
mermaid: true
---
istio Observability 살펴보기
<!--more-->


### 들어가며


이번 주차에서는 Istio Obervability에 대해 알아본다. istio는 파드의 sidecar로 붙어 모든 서비스 네트워크에 관여한다. 그렇기에 istio 기반으로 네트워크 모니터링 시스템을 구축한다면 모든 서비스 네트워크에 대한 관찰 가능성을 높일 수 있다.


운영을 하다보면 모니터링이 정말 중요하다고 느끼는데, 여기서는 Istio의 주요 지표에 대해 알아보며 특정 상황에 어떤 지표를 봐야하는지 살펴볼 예정이다. 


*아래에서 사용되는 도식화는 스터디원 분이 직접 그려주셨습니다. 감사합니다 (_ _)


### 실습 구성


실습은 이전 포스팅과 동일하게 kind(`v1.23.17`), istio(`v1.17.8`) & sidecar mode로 구성한다.


리소스는 아래와 같이 배포한다.


```bash
# catalog
kubectl apply -f services/catalog/kubernetes/catalog.yaml -n istioinaction

# webapp
kubectl apply -f services/webapp/kubernetes/webapp.yaml -n istioinaction

# gateway, virtualservice 설정
kubectl apply -f services/webapp/istio/webapp-catalog-gw-vs.yaml -n istioinaction
```


실습환경을 도식화하면 아래와 같다.


![image.png](/assets/img/post/Istio%20Observability(Metric)/1.png)


## Istio metric


Istio에서 실제 프록시로 사용하고 있는 것은 Envoy이다. 그렇기에 관련 정보도 Envoy를 통해 가져온다. Envoy는 정말 많은 정보를 가지고 있지만, 과도한 로깅 및 지표 수집은 전체 시스템에 영향을 끼칠 수 있다. 그렇기에 Istio에서는 아래에서 살펴볼 표준 메트릭만 가져온다.


Istio에서는 초당 요청 수, 요청 처리에 걸리는 시간(백분위수로 구분), 실패한 요청 수 등의 정보를 가져올 수 있으며 새 메트릭을 동적으로 추가할 수 있다.


### data plane metric


#### istio 표준 metrics


관련 링크: [https://istio.io/latest/docs/reference/config/metrics/](https://istio.io/latest/docs/reference/config/metrics/)


HTTP 관련 지표는 아래와 같다. 


| istio_requests_total                | Counter      | 처리된 **전체 요청 수**          |
| ----------------------------------- | ------------ | ------------------------ |
| istio_request_duration_milliseconds | Distribution | 요청 처리에 걸린 **시간(ms 단위)**  |
| istio_request_bytes                 | Distribution | HTTP **요청 본문의 크기**       |
| istio_response_bytes                | Distribution | HTTP **응답 본문의 크기**       |
| istio_request_messages_total        | Counter      | 클라이언트가 보낸 **gRPC 메시지 수** |
| istio_response_messages_total       | Counter      | 서버가 보낸 **gRPC 메시지 수**    |


쿼리를 통해 엔보이(Proxy)가 보관하는 정보를 확인할 수 있다. 위에서 확인한 표준 메트릭도 보인다. 이들 중 대부분은 기본적으로 삭제된다.


```bash
kubectl exec -it deploy/catalog -c istio-proxy -n istioinaction -- curl localhost:15000/stats

cluster_manager.cds.version_text: "2025-05-03T12:39:09Z/11"
listener_manager.lds.version_text: "2025-05-03T12:39:09Z/11"
cluster.xds-grpc.assignment_stale: 0
cluster.xds-grpc.assignment_timeout_received: 0
cluster.xds-grpc.bind_errors: 0
cluster.xds-grpc.circuit_breakers.default.cx_open: 0
...
cluster.xds-grpc.circuit_breakers.high.rq_retry_open: 0
cluster.xds-grpc.default.total_match_count: 1
cluster.xds-grpc.http2.deferred_stream_close: 0
...
cluster.xds-grpc.http2.tx_flush_timeout: 0
cluster.xds-grpc.http2.tx_reset: 0
cluster.xds-grpc.internal.upstream_rq_200: 3
cluster.xds-grpc.internal.upstream_rq_2xx: 3
cluster.xds-grpc.internal.upstream_rq_completed: 3
cluster.xds-grpc.lb_healthy_panic: 0
cluster.xds-grpc.lb_local_cluster_not_ok: 0
cluster.xds-grpc.lb_recalculate_zone_structures: 0
...
cluster.xds-grpc.max_host_weight: 0
cluster.xds-grpc.membership_change: 1
cluster.xds-grpc.membership_degraded: 0
cluster.xds-grpc.membership_excluded: 0
cluster.xds-grpc.membership_healthy: 1
cluster.xds-grpc.membership_total: 1
cluster.xds-grpc.original_dst_host_invalid: 0
cluster.xds-grpc.retry_or_shadow_abandoned: 0
cluster.xds-grpc.update_attempt: 0
cluster.xds-grpc.update_empty: 0
cluster.xds-grpc.update_failure: 0
cluster.xds-grpc.update_no_rebuild: 0
cluster.xds-grpc.update_success: 0
cluster.xds-grpc.upstream_cx_active: 1
cluster.xds-grpc.upstream_cx_close_notify: 0
cluster.xds-grpc.upstream_cx_connect_attempts_exceeded: 0
cluster.xds-grpc.upstream_cx_connect_fail: 0
cluster.xds-grpc.upstream_cx_connect_timeout: 0
cluster.xds-grpc.upstream_cx_connect_with_0_rtt: 0
cluster.xds-grpc.upstream_cx_destroy: 2
cluster.xds-grpc.upstream_cx_destroy_local: 2
...
cluster.xds-grpc.upstream_rq_per_try_timeout: 0
cluster.xds-grpc.upstream_rq_retry: 0
cluster.xds-grpc.upstream_rq_retry_backoff_exponential: 0
cluster.xds-grpc.upstream_rq_retry_backoff_ratelimited: 0
cluster.xds-grpc.upstream_rq_retry_limit_exceeded: 0
cluster.xds-grpc.upstream_rq_retry_overflow: 0
cluster.xds-grpc.upstream_rq_retry_success: 0
cluster.xds-grpc.upstream_rq_rx_reset: 0
cluster.xds-grpc.upstream_rq_timeout: 0
cluster.xds-grpc.upstream_rq_total: 3
cluster.xds-grpc.upstream_rq_tx_reset: 2
cluster.xds-grpc.version: 0
cluster_manager.active_clusters: 21
cluster_manager.cds.init_fetch_timeout: 0
cluster_manager.cds.update_attempt: 9
...
istiocustom.istio_build.component.proxy.tag.1.17.8: 1
istiocustom.istio_requests_total.reporter.destination.source_workload.webapp.source_canonical_service.webapp.source_canonical_revision.latest.source_workload_namespace.istioinaction.source_principal.spiffe://cluster.local/ns/istioinaction/sa/webapp.source_app.webapp.source_version.unknown.source_cluster.Kubernetes.destination_workload.catalog.destination_workload_namespace.istioinaction.destination_principal.spiffe://cluster.local/ns/istioinaction/sa/catalog.destination_app.catalog.destination_version.v1.destination_service.catalog.istioinaction.svc.cluster.local.destination_canonical_service.catalog.destination_canonical_revision.v1.destination_service_name.catalog.destination_service_namespace.istioinaction.destination_cluster.Kubernetes.request_protocol.http.response_code.200.grpc_response_status.response_flags.-.connection_security_policy.mutual_tls: 2
listener_manager.lds.init_fetch_timeout: 0
...
server.envoy_bug_failures: 0
server.hot_restart_epoch: 0
...
wasm.remote_load_cache_negative_hits: 0
wasm.remote_load_fetch_failures: 0
wasm.remote_load_fetch_successes: 0
...
listener_manager.lds.update_duration: P0(nan,0) P25(nan,0) P50(nan,0) P75(nan,1.05) P90(nan,8.04) P95(nan,8.07) P99(nan,8.094) P99.5(nan,8.097) P99.9(nan,8.0994) P100(nan,8.1)
server.initialization_time_ms: P0(nan,130) P25(nan,132.5) P50(nan,135) P75(nan,137.5) P90(nan,139) P95(nan,139.5) P99(nan,139.9) P99.5(nan,139.95) P99.9(nan,139.99) P100(nan,140)
```


#### 엔보이 통계 추가 설정


Istio 표준 메트릭보다 더 많은 정보를 확인이 필요한 경우 엔보이에서 더 많은 지표를 가져올 수 있다. 


설정하는 방법은 크게 2가지 방법이 있다.


##### IstioOperator


하지만, 아래와 같은 경우는 전체 서비스에 대해 적용하는 것이므로, 과도한 로깅으로 전체 시스템에 영향을 끼칠 수 있다.


```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: control-plane
spec:
  profile: demo
  meshConfig:
    defaultConfig: # Defines the default proxy configuration for all services
      proxyStatsMatcher: # Customizes the reported metrics
        inclusionPrefixes: # Metrics matching the prefix will be reported alongside the default ones.
        - "cluster.outbound|80||catalog.istioinaction"
```


##### Annotation


워크로드 단위로 설정하는 방식으로, 주석을 통해 메트릭을 지정한다.


아래의 의미를 확인해보면, `inclusionPrefixes` = 해당 prefix로 시작하는 통계 항목도 Envoy에서 가져와 수집하도록 진행한다. 즉 아래에서는 80포트를 통해 catalog.istioinaction로 향하는 아웃바운드 트래픽을 모두 수집한다.


```bash
cat ch7/webapp-deployment-stats-inclusion.yaml
...
      annotations:
        proxy.istio.io/config: |-
          proxyStatsMatcher:
            inclusionPrefixes:
            - "cluster.outbound|80||catalog.istioinaction"
      labels:
        app: webapp
```


이제 세팅을 적용하고, istio proxy의 통계지표를 확인해보자. `cluster.outbound|80||catalog.istioinaction` 로 향하는 트래픽을 수집할 수 있다.


```bash
kubectl exec -it deploy/webapp -c istio-proxy -n istioinaction -- curl localhost:15000/stats | grep catalog

cluster.outbound|80||catalog.istioinaction.svc.cluster.local.version_text: "2025-05-03T12:39:09Z/11"
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.assignment_stale: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.assignment_timeout_received: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.bind_errors: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.cx_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.cx_pool_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.remaining_cx: 4294967294
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.remaining_cx_pools: 18446744073709551614
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.remaining_pending: 4294967295
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.remaining_retries: 4294967295
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.remaining_rq: 4294967295
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.rq_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.default.rq_retry_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.high.cx_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.high.cx_pool_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.high.rq_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.circuit_breakers.high.rq_retry_open: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.client_ssl_socket_factory.downstream_context_secrets_not_ready: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.client_ssl_socket_factory.ssl_context_update_by_sds: 2
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.client_ssl_socket_factory.upstream_context_secrets_not_ready: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.default.total_match_count: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.http1.dropped_headers_with_underscores: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.http1.metadata_not_supported_error: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.http1.requests_rejected_with_underscores_in_headers: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.http1.response_flood: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.init_fetch_timeout: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_200: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_2xx: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_completed: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_healthy_panic: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_local_cluster_not_ok: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_recalculate_zone_structures: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_subsets_active: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_subsets_created: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_subsets_fallback: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_subsets_fallback_panic: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_subsets_removed: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_subsets_selected: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_zone_cluster_too_small: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_zone_no_capacity_left: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_zone_number_differs: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_zone_routing_all_directly: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_zone_routing_cross_zone: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.lb_zone_routing_sampled: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.max_host_weight: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.membership_change: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.membership_degraded: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.membership_excluded: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.membership_healthy: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.membership_total: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.metadata_exchange.alpn_protocol_found: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.metadata_exchange.alpn_protocol_not_found: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.metadata_exchange.header_not_found: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.metadata_exchange.initial_header_not_found: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.metadata_exchange.metadata_added: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.original_dst_host_invalid: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.retry_or_shadow_abandoned: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.ciphers.TLS_AES_128_GCM_SHA256: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.connection_error: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.curves.X25519: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.fail_verify_cert_hash: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.fail_verify_error: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.fail_verify_no_cert: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.fail_verify_san: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.handshake: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.no_certificate: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.ocsp_staple_failed: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.ocsp_staple_omitted: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.ocsp_staple_requests: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.ocsp_staple_responses: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.session_reused: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.sigalgs.rsa_pss_rsae_sha256: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.ssl.versions.TLSv1.3: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.tlsMode-disabled.total_match_count: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.tlsMode-istio.total_match_count: 2
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_attempt: 3
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_empty: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_failure: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_no_rebuild: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_rejected: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_success: 2
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_time: 1746281155385
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_active: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_close_notify: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_connect_attempts_exceeded: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_connect_fail: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_connect_timeout: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_connect_with_0_rtt: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_destroy: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_destroy_local: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_destroy_local_with_active_rq: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_destroy_remote: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_destroy_remote_with_active_rq: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_destroy_with_active_rq: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_http1_total: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_http2_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_http3_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_idle_timeout: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_max_duration_reached: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_max_requests: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_none_healthy: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_overflow: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_pool_overflow: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_protocol_error: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_rx_bytes_buffered: 1709
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_rx_bytes_total: 1709
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_total: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_tx_bytes_buffered: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_tx_bytes_total: 1284
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_flow_control_backed_up_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_flow_control_drained_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_flow_control_paused_reading_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_flow_control_resumed_reading_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_http3_broken: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_internal_redirect_failed_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_internal_redirect_succeeded_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_0rtt: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_200: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_2xx: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_active: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_cancelled: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_completed: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_maintenance_mode: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_max_duration_reached: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_pending_overflow: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_pending_total: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_per_try_idle_timeout: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_per_try_timeout: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_retry: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_retry_backoff_exponential: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_retry_backoff_ratelimited: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_retry_limit_exceeded: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_retry_overflow: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_retry_success: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_rx_reset: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_timeout: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_total: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_tx_reset: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.version: 12825535957486651501
istiocustom.istio_requests_total.reporter.source.source_workload.webapp.source_canonical_service.webapp.source_canonical_revision.latest.source_workload_namespace.istioinaction.source_principal.spiffe://cluster.local/ns/istioinaction/sa/webapp.source_app.webapp.source_version.source_cluster.Kubernetes.destination_workload.catalog.destination_workload_namespace.istioinaction.destination_principal.spiffe://cluster.local/ns/istioinaction/sa/catalog.destination_app.catalog.destination_version.v1.destination_service.catalog.istioinaction.svc.cluster.local.destination_canonical_service.catalog.destination_canonical_revision.v1.destination_service_name.catalog.destination_service_namespace.istioinaction.destination_cluster.Kubernetes.request_protocol.http.response_code.200.grpc_response_status.response_flags.-.connection_security_policy.unknown: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_time: P0(nan,11) P25(nan,11.25) P50(nan,11.5) P75(nan,11.75) P90(nan,11.9) P95(nan,11.95) P99(nan,11.99) P99.5(nan,11.995) P99.9(nan,11.999) P100(nan,12)
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.update_duration: P0(nan,0) P25(nan,0) P50(nan,0) P75(nan,0) P90(nan,0) P95(nan,0) P99(nan,0) P99.5(nan,0) P99.9(nan,0) P100(nan,0)
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_connect_ms: P0(nan,11) P25(nan,11.25) P50(nan,11.5) P75(nan,11.75) P90(nan,11.9) P95(nan,11.95) P99(nan,11.99) P99.5(nan,11.995) P99.9(nan,11.999) P100(nan,12)
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_cx_length_ms: No recorded values
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_rq_time: P0(nan,11) P25(nan,11.25) P50(nan,11.5) P75(nan,11.75) P90(nan,11.9) P95(nan,11.95) P99(nan,11.99) P99.5(nan,11.995) P99.9(nan,11.999) P100(nan,12)
istiocustom.istio_request_bytes.reporter.source.source_workload.webapp.source_canonical_service.webapp.source_canonical_revision.latest.source_workload_namespace.istioinaction.source_principal.spiffe://cluster.local/ns/istioinaction/sa/webapp.source_app.webapp.source_version.source_cluster.Kubernetes.destination_workload.catalog.destination_workload_namespace.istioinaction.destination_principal.spiffe://cluster.local/ns/istioinaction/sa/catalog.destination_app.catalog.destination_version.v1.destination_service.catalog.istioinaction.svc.cluster.local.destination_canonical_service.catalog.destination_canonical_revision.v1.destination_service_name.catalog.destination_service_namespace.istioinaction.destination_cluster.Kubernetes.request_protocol.http.response_code.200.grpc_response_status.response_flags.-.connection_security_policy.unknown: P0(nan,630) P25(nan,632.5) P50(nan,635) P75(nan,637.5) P90(nan,639) P95(nan,639.5) P99(nan,639.9) P99.5(nan,639.95) P99.9(nan,639.99) P100(nan,640)
istiocustom.istio_request_duration_milliseconds.reporter.source.source_workload.webapp.source_canonical_service.webapp.source_canonical_revision.latest.source_workload_namespace.istioinaction.source_principal.spiffe://cluster.local/ns/istioinaction/sa/webapp.source_app.webapp.source_version.source_cluster.Kubernetes.destination_workload.catalog.destination_workload_namespace.istioinaction.destination_principal.spiffe://cluster.local/ns/istioinaction/sa/catalog.destination_app.catalog.destination_version.v1.destination_service.catalog.istioinaction.svc.cluster.local.destination_canonical_service.catalog.destination_canonical_revision.v1.destination_service_name.catalog.destination_service_namespace.istioinaction.destination_cluster.Kubernetes.request_protocol.http.response_code.200.grpc_response_status.response_flags.-.connection_security_policy.unknown: P0(nan,16) P25(nan,16.25) P50(nan,16.5) P75(nan,16.75) P90(nan,16.9) P95(nan,16.95) P99(nan,16.99) P99.5(nan,16.995) P99.9(nan,16.999) P100(nan,17)
istiocustom.istio_response_bytes.reporter.source.source_workload.webapp.source_canonical_service.webapp.source_canonical_revision.latest.source_workload_namespace.istioinaction.source_principal.spiffe://cluster.local/ns/istioinaction/sa/webapp.source_app.webapp.source_version.source_cluster.Kubernetes.destination_workload.catalog.destination_workload_namespace.istioinaction.destination_principal.spiffe://cluster.local/ns/istioinaction/sa/catalog.destination_app.catalog.destination_version.v1.destination_service.catalog.istioinaction.svc.cluster.local.destination_canonical_service.catalog.destination_canonical_revision.v1.destination_service_name.catalog.destination_service_namespace.istioinaction.destination_cluster.Kubernetes.request_protocol.http.response_code.200.grpc_response_status.response_flags.-.connection_security_policy.unknown: P0(nan,860) P25(nan,862.5) P50(nan,865) P75(nan,867.5) P90(nan,869) P95(nan,869.5) P99(nan,869.9) P99.5(nan,869.95) P99.9(nan,869.99) P100(nan,870)
```


추가로 istio에서는 인그레스를 통해 들어온 트래픽 = 외부, 나머지는 내부로 인식한다. 아래를 통해 내부에서 시작한 트래픽과 관련된 지표도 볼 수 있다.


```bash
kubectl exec -it deploy/webapp -c istio-proxy -n istioinaction -- curl localhost:15000/stats | grep catalog | grep internal

cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_200: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_2xx: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_completed: 1
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_internal_redirect_failed_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.upstream_internal_redirect_succeeded_total: 0
cluster.outbound|80||catalog.istioinaction.svc.cluster.local.internal.upstream_rq_time: P0(nan,11) P25(nan,11.25) P50(nan,11.5) P75(nan,11.75) P90(nan,11.9) P95(nan,11.95) P99(nan,11.99) P99.5(nan,11.995) P99.9(nan,11.999) P100(nan,12)
```


### control plane metric


기존에는 data plane, 실제 서비스에서 오고가는 트래픽에 대한 지표를 알아봤다면 여기서는 istiod 동작과 관련된 지표를 알아본다. ex) 프록시 동기화 횟수, 인증서 발급 정보 등

- proxy sync 관련
	- xDS API의 **업데이트 횟수**
	- 동기화 시간
- 인증서 관련
	- CSR(Certificate Signing Request 인증서 발급 요청)
	- Citadel : Istio 보안 관련 지표

istiod는 Pilot, Galley, Citadel 3개의 요소가 존재한다. Pilot은 프록시 라우팅 규칙을 관리하며, Galley는 Endpoint 갱신 등 K8s와 상호작용하며, Citadel은 보안 및 암호화 관련 기능을 담당한다. 아래의 명령어를 통해 포트를 확인한다.


```bash
kubectl exec -it deploy/istiod -n istio-system -- netstat -tnl

Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:9876          0.0.0.0:*               LISTEN     
tcp6       0      0 :::8080                 :::*                    LISTEN     
tcp6       0      0 :::15017                :::*                    LISTEN     
tcp6       0      0 :::15012                :::*                    LISTEN     
tcp6       0      0 :::15014                :::*                    LISTEN     
tcp6       0      0 :::15010                :::*                    LISTEN     
```


![image.png](/assets/img/post/Istio%20Observability(Metric)/2.png)


#### 관련 지표 확인 명령어


아래의 명령어를 통해 istiod의 모든 지표를 확인할 수 있다. 이제 주요 지표를 하나씩 살펴보자.


```bash
kubectl exec -it -n istio-system deploy/istiod -n istio-system -- curl localhost:15014/metrics
```

- 프록시 동기화 관련 지표

`pilot_proxy_convergence_time_bucket{le="0.1"}: 27` 이벤트 27개를 동기화하는 데, 0.1초 이하의 시간이 걸렸다는 의미이다.


```bash
kubectl exec -it -n istio-system deploy/istiod -n istio-system -- curl localhost:15014/metrics | grep convergence 

# HELP pilot_proxy_convergence_time Delay in seconds between config change and a proxy receiving all required configuration.
# TYPE pilot_proxy_convergence_time histogram
pilot_proxy_convergence_time_bucket{le="0.1"} 27
pilot_proxy_convergence_time_bucket{le="0.5"} 27
pilot_proxy_convergence_time_bucket{le="1"} 27
pilot_proxy_convergence_time_bucket{le="3"} 27
pilot_proxy_convergence_time_bucket{le="5"} 27
pilot_proxy_convergence_time_bucket{le="10"} 27
pilot_proxy_convergence_time_bucket{le="20"} 27
pilot_proxy_convergence_time_bucket{le="30"} 27
pilot_proxy_convergence_time_bucket{le="+Inf"} 27
pilot_proxy_convergence_time_sum 0.020522709000000004
pilot_proxy_convergence_time_count 27
```

- xDS API

xDS는 Envoy 설정을 업데이트하는 API이다. 관련 API 호출 횟수를 확인해보자. 각 Envoy 영역에 대한 업데이트 횟수를 볼 수 있다.


```bash
kubectl exec -it -n istio-system deploy/istiod -n istio-system -- curl localhost:15014/metrics | grep pilot_xds_pushes

# HELP pilot_xds_pushes Pilot build and send errors for lds, rds, cds and eds.
# TYPE pilot_xds_pushes counter
pilot_xds_pushes{type="cds"} 26
pilot_xds_pushes{type="eds"} 38
pilot_xds_pushes{type="lds"} 26
pilot_xds_pushes{type="rds"} 20
```


### 프로메테우스


프로메테우스는 대표적인 모니터링 및 알람 시스템으로 지표를 수집한다. 프로메테우스는 지표를 TSDB(시계열 데이터베이스) 형태로 저장하며, PromQL을 통해 쿼리를 진행한다.


또한, 데이터를 가져오는 방식으로는 에이전트가 메트릭을 밀어넣기(push)가 아닌 메트릭을 당겨온다(pull)고 한다. 


![image.png](/assets/img/post/Istio%20Observability(Metric)/3.png)


출처: [https://prometheus.io/docs/introduction/overview/](https://prometheus.io/docs/introduction/overview/)


istio는 이런 프로메테우스가 지표를 잘 가져오도록 프로메테우스 메트릭을 기본적으로 내장하여 노출한다.


```bash
kubectl exec -it deploy/webapp -c istio-proxy -n istioinaction -- curl localhost:15090/stats/prometheus

# TYPE envoy_cluster_assignment_stale counter
envoy_cluster_assignment_stale{cluster_name="outbound|80||catalog.istioinaction.svc.cluster.local"} 0
envoy_cluster_assignment_stale{cluster_name="xds-grpc"} 0
# TYPE envoy_cluster_assignment_timeout_received counter
envoy_cluster_assignment_timeout_received{cluster_name="outbound|80||catalog.istioinaction.svc.cluster.local"} 0
envoy_cluster_assignment_timeout_received{cluster_name="xds-grpc"} 0
# TYPE envoy_cluster_bind_errors counter
envoy_cluster_bind_errors{cluster_name="outbound|80||catalog.istioinaction.svc.cluster.local"} 0
envoy_cluster_bind_errors{cluster_name="xds-grpc"} 0
# TYPE envoy_cluster_client_ssl_socket_factory_downstream_context_secrets_not_ready counter
envoy_cluster_client_ssl_socket_factory_downstream_context_secrets_not_ready{cluster_name="outbound|80||catalog.istioinaction.svc.cluster.local"} 0
# TYPE envoy_cluster_client_ssl_socket_factory_ssl_context_update_by_sds counter
envoy_cluster_client_ssl_socket_factory_ssl_context_update_by_sds{cluster_name="outbound|80||catalog.istioinaction.svc.cluster.local"} 2
# TYPE envoy_cluster_client_ssl_socket_factory_upstream_context_secrets_not_ready counter
...
envoy_server_initialization_time_ms_count{} 1
```


#### 설치


helm repo 추가


```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```


helm valuse 팡리


```bash
cat << EOF > prom-values-2.yaml
prometheusOperator:
  tls:
    enabled: false
  admissionWebhooks:
    patch:
      enabled: false

prometheus:
  service:
    type: NodePort
    nodePort: 30001
    
grafana:
  service:
    type: NodePort
    nodePort: 30002
EOF
```


이제 프로메테우스 네임스페이스를 만들고, 배포를 진행한다.


```bash
kubectl create ns prometheus

namespace/prometheus created
```


helm을 통해 프로메테우스를 배포한다.


```bash
helm install prom prometheus-community/kube-prometheus-stack --version 13.13.1 \
-n prometheus -f ch7/prom-values.yaml -f prom-values-2.yaml
NAME: prom
LAST DEPLOYED: Sun May  4 00:06:02 2025
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace prometheus get pods -l "release=prom"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```


실제 30001 포트에 접속하여 배포된 프로메테우스를 확인한다.


![image.png](/assets/img/post/Istio%20Observability(Metric)/4.png)


#### istio 설정 진행


프로메테우스 오퍼레이터의 커스텀 리소스인 `ServiceMonitor` 와 `PodMonitor` 를 사용하여 이스티오에서 메트릭을 수집한다.


관련 커스텀 리소스의 동작방식은 아래의 도식화에서 살펴볼 수 있다. selector 및 match 방식으로 진행하여 실제 Service와 매핑된 서비스 파드의 트래픽 리소스를 가져온다.


![image.png](/assets/img/post/Istio%20Observability(Metric)/5.png)


출처: [https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/img/custom-metrics-elements.png](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/img/custom-metrics-elements.png)


##### control plane 지표 수집


아래의 명령어를 통해 istiod 서비스에서 monitoring 관련 엔드포인트를 확인할 수 있다. 


```bash
kubectl describe svc istiod -n istio-system

Name:              istiod
Namespace:         istio-system
Labels:            app=istiod
                   install.operator.istio.io/owning-resource=unknown
                   install.operator.istio.io/owning-resource-namespace=istio-system
                   istio=pilot
                   istio.io/rev=default
                   operator.istio.io/component=Pilot
                   operator.istio.io/managed=Reconcile
                   operator.istio.io/version=1.17.8
                   release=istio
Annotations:       <none>
Selector:          app=istiod,istio=pilot
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.200.1.86
IPs:               10.200.1.86
Port:              grpc-xds  15010/TCP
TargetPort:        15010/TCP
Endpoints:         10.10.0.2:15010
Port:              https-dns  15012/TCP
TargetPort:        15012/TCP
Endpoints:         10.10.0.2:15012
Port:              https-webhook  443/TCP
TargetPort:        15017/TCP
Endpoints:         10.10.0.2:15017
Port:              http-monitoring  15014/TCP
TargetPort:        15014/TCP
Endpoints:         10.10.0.2:15014
Session Affinity:  None
Events:            <none>
```


컨트롤 플레인 지표 수집을 위해, 모니터링과 관련된 Service Monitor 리소스를 배포한다.


```bash
# cat ch7/service-monitor-cp.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-component-monitor
  namespace: prometheus
  labels:
    monitoring: istio-components
    release: prom
spec:
  jobLabel: istio
  targetLabels: [app]
  selector:
    matchExpressions:
    - {key: istio, operator: In, values: [pilot]}
  namespaceSelector:
    any: true
  endpoints:
  - port: http-monitoring # 15014
    interval: 15s
```


수집될 지표의 정보는 아래의 명령어로도 확인 가능하다.


```bash
kubectl exec -it netshoot -- curl -s istiod.istio-system:15014/metrics
```


직접 웹에 접속하여 확인해보면, 아래와 같이 istiod에 대한 지표를 확인할 수 있다.


![image.png](/assets/img/post/Istio%20Observability(Metric)/6.png)


##### data plane 지표 수집


이제 아래와 같이 PodMonitor를 배포하여, istio-proxy 컨테이너가 존재하는 모든 파드에 대한 메트릭을 수집한다.


Pod 모니터링 관련 리소스를 배포한다. 리소스의 내용을 요약해보면 아래와 같다.

- target: `istio-proxy` 컨테이너
- 수집 path: `/stats/prometheus`

```bash
cat ch7/pod-monitor-dp.yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: envoy-stats-monitor
  namespace: prometheus
  labels:
    monitoring: istio-proxies
    release: prom
spec:
  selector:
    matchExpressions:
    - {key: istio-prometheus-ignore, operator: DoesNotExist}
  namespaceSelector:
    any: true
  jobLabel: envoy-stats
  podMetricsEndpoints:
  - path: /stats/prometheus
    interval: 15s
    relabelings:
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_container_name]
      regex: "istio-proxy"
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_annotationpresent_prometheus_io_scrape]
    - sourceLabels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      targetLabel: __address__
    - action: labeldrop
      regex: "__meta_kubernetes_pod_label_(.+)"
    - sourceLabels: [__meta_kubernetes_namespace]
      action: replace
      targetLabel: namespace
    - sourceLabels: [__meta_kubernetes_pod_name]
      action: replace
      targetLabel: pod_name%                                                                                       
```


테스트를 위해 터미널에선 반복 접속을 진행한다.


```bash
while true; do curl -s http://webapp.istioinaction.io:30000/api/catalog ; date "+%Y-%m-%d %H:%M:%S" ; sleep 1; echo; done
```


이제 실제 istio_request_total을 확인해보면, 아래와 같다. PodMonitory interval 기간인 15초마다 새로고침하면 지표값이 증가하는 것을 확인할 수 있다.


![image.png](/assets/img/post/Istio%20Observability(Metric)/7.png)


### Custom


지금까진 Istio 표준 메트릭에 대한 수집 방식에 대해 알아봤다. 여기서는 istio 표준은 아니지만 엔보이에서 제공하는 매트릭을 수집하는 방법에 대해 알아본다. 


#### 기존 메트릭 수정

- istio operator 기존 설정 확인하기

```bash
kubectl get istiooperator installed-state -n istio-system -o yaml | grep -E "prometheus:|telemetry:" -A2

    telemetry:
      enabled: true
      v2:
--
        prometheus:
          enabled: true
          wasmEnabled: false
```

- 새로운 설정 배포하기

아래의 방법은 request_protocol을 디멘션을 제거하고, 호출한 쪽의 mesh id와 istio version을 기록하도록 설정하는 방법이다.


```bash
cat ch7/metrics/istio-operator-new-dimensions.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  values:
    telemetry:
      v2:
        prometheus:
          configOverride:
            inboundSidecar:
              metrics:
              - name: requests_total
                dimensions:
                  upstream_proxy_version: upstream_peer.istio_version
                  source_mesh_id: node.metadata['MESH_ID']
                tags_to_remove:
                - request_protocol
            outboundSidecar:
              metrics:
              - name: requests_total
                dimensions:
                  upstream_proxy_version: upstream_peer.istio_version
                  source_mesh_id: node.metadata['MESH_ID']
                tags_to_remove:
                - request_protocol
            gateway:
              metrics:
              - name: requests_total
                dimensions:
                  upstream_proxy_version: upstream_peer.istio_version
                  source_mesh_id: node.metadata['MESH_ID']
                tags_to_remove:
                - request_protocol
```


노드에 접속하여 위의 operator 설정을 진행한다.


```bash
docker exec -it myk8s-control-plane bash
---
istioctl verify-install -f istio-operator-new-dimensions.yaml # 리소스별로 적용결과를 출력
istioctl install -f istio-operator-new-dimensions.yaml -y
```


적용 전의 istio_request_total의 지표값을 확인해보자.


```bash
kubectl -n istioinaction exec -it deploy/webapp -c istio-proxy \
-- curl localhost:15000/stats/prometheus | grep istio_requests_total
# TYPE istio_requests_total counter
istio_requests_total{reporter="destination",source_workload="istio-ingressgateway",source_canonical_service="istio-ingressgateway",source_canonical_revision="latest",source_workload_namespace="istio-system",source_principal="spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account",source_app="istio-ingressgateway",source_version="unknown",source_cluster="Kubernetes",destination_workload="webapp",destination_workload_namespace="istioinaction",destination_principal="spiffe://cluster.local/ns/istioinaction/sa/webapp",destination_app="webapp",destination_version="",destination_service="webapp.istioinaction.svc.cluster.local",destination_canonical_service="webapp",destination_canonical_revision="latest",destination_service_name="webapp",destination_service_namespace="istioinaction",destination_cluster="Kubernetes",request_protocol="http",response_code="200",grpc_response_status="",response_flags="-",connection_security_policy="mutual_tls"} 310
istio_requests_total{reporter="source",source_workload="webapp",source_canonical_service="webapp",source_canonical_revision="latest",source_workload_namespace="istioinaction",source_principal="spiffe://cluster.local/ns/istioinaction/sa/webapp",source_app="webapp",source_version="",source_cluster="Kubernetes",destination_workload="catalog",destination_workload_namespace="istioinaction",destination_principal="spiffe://cluster.local/ns/istioinaction/sa/catalog",destination_app="catalog",destination_version="v1",destination_service="catalog.istioinaction.svc.cluster.local",destination_canonical_service="catalog",destination_canonical_revision="v1",destination_service_name="catalog",destination_service_namespace="istioinaction",destination_cluster="Kubernetes",request_protocol="http",response_code="200",grpc_response_status="",response_flags="-",connection_security_policy="unknown"} 310
```


적용 후의 istio_request_total의 지표값을 보면, `source_mesh_id="cluster.local”`와 `upstream_proxy_version="1.17.8”` 가 추가된 것을 확인할 수 있다.


```bash
kubectl -n istioinaction exec -it deploy/webapp -c istio-proxy \
-- curl localhost:15000/stats/prometheus | grep istio_requests_total
# TYPE istio_requests_total counter
istio_requests_total{reporter="destination",source_workload="istio-ingressgateway",source_canonical_service="istio-ingressgateway",source_canonical_revision="latest",source_workload_namespace="istio-system",source_principal="spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account",source_app="istio-ingressgateway",source_version="unknown",source_cluster="Kubernetes",destination_workload="webapp",destination_workload_namespace="istioinaction",destination_principal="spiffe://cluster.local/ns/istioinaction/sa/webapp",destination_app="webapp",destination_version="",destination_service="webapp.istioinaction.svc.cluster.local",destination_canonical_service="webapp",destination_canonical_revision="latest",destination_service_name="webapp",destination_service_namespace="istioinaction",destination_cluster="Kubernetes",response_code="200",grpc_response_status="",response_flags="-",connection_security_policy="mutual_tls",source_mesh_id="cluster.local",upstream_proxy_version="unknown"} 116
istio_requests_total{reporter="source",source_workload="webapp",source_canonical_service="webapp",source_canonical_revision="latest",source_workload_namespace="istioinaction",source_principal="spiffe://cluster.local/ns/istioinaction/sa/webapp",source_app="webapp",source_version="",source_cluster="Kubernetes",destination_workload="catalog",destination_workload_namespace="istioinaction",destination_principal="spiffe://cluster.local/ns/istioinaction/sa/catalog",destination_app="catalog",destination_version="v1",destination_service="catalog.istioinaction.svc.cluster.local",destination_canonical_service="catalog",destination_canonical_revision="v1",destination_service_name="catalog",destination_service_namespace="istioinaction",destination_cluster="Kubernetes",response_code="200",grpc_response_status="",response_flags="-",connection_security_policy="unknown",source_mesh_id="cluster.local",upstream_proxy_version="1.17.8"} 118
```


#### 새로운 메트릭 추가


새로운 메트릭을 추가하는 방법은, 아래와 같이 새로운 메트릭을 정의하면 된다. 아래의 지표는 요청으로 온 HTTP 메서드가 `GET`으로 시작할 경우마다 1을 카운트하는 지표이다.


```yaml
# cat ch7/metrics/istio-operator-new-metric.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  values:
    telemetry:
      v2:
        prometheus:
          configOverride:
            inboundSidecar:
              definitions:
              - name: get_calls
                type: COUNTER
                value: "(request.method.startsWith('GET') ? 1 : 0)"
            outboundSidecar:
              definitions:
              - name: get_calls
                type: COUNTER
                value: "(request.method.startsWith('GET') ? 1 : 0)"
            gateway:
              definitions:
              - name: get_calls
                type: COUNTER
                value: "(request.method.startsWith('GET') ? 1 : 0)"
```


이를 위해선 또 control plane node에 접속하여 istio operator 설정 적용이 필요하다.


```bash
docker exec -it myk8s-control-plane bash
---
istioctl verify-install -f istio-operator-new-metric.yaml # 리소스별로 적용결과를 출력
istioctl install -f istio-operator-new-metric.yaml -y
```


이제 관련하여 파드에 아래와 같은 어노테이션을 붙여 해당 파드에서 위에서 설정한 지표 수집이 진행되도록 설정한다.


```bash
cat ch7/metrics/webapp-deployment-new-metric.yaml | grep -A4 annotations:
      annotations:
        proxy.istio.io/config: |-
          proxyStatsMatcher:
            inclusionPrefixes:
            - "istio_get_calls"
```


이제 확인해보면, 아래와 같이 우리가 요청한 횟수만큼 지표가 찍히는 것을 볼 수 있다.


```bash
kubectl -n istioinaction exec -it deploy/webapp -c istio-proxy -- curl localhost:15000/stats/prometheus | grep istio_get_calls

# TYPE istio_get_calls counter
istio_get_calls{} 22
```


해당 지표는 프로메테우스 UI에서도 확인할 수 있다.


![image.png](/assets/img/post/Istio%20Observability(Metric)/8.png)


## 마치며


이번 포스팅에서는 istio의 Metric에 대해 살펴봤다. 실제 서비스 트래픽에 대한 지표 수집과 istiod에 대한 지표수집이 가능하다. 또, istio는 기본적으로 프로메테우스 메트릭을 노출하는 형태이며, istio가 기본적으로 제공하는 표준 metric이 존재한다. 하지만, envoy에는 더 많은 정보를 가지고 있고 이에 대한 추가 정보가 필요하다면 istio operator와 각 pod의 annotation 설정으로 지표를 커스터마이징할 수 있다.


다음 포스팅에서는 이 지표를 시각화하는 대시보드에 대해 알아볼 예정이다.

