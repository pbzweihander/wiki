# 쿠버네티스 NodePort 서비스

쿠버네티스 서비스에는 ClusterIP, NodePort, LoadBalancer 등이 있는데, 이 중에서
NodePort는 그 클러스터에서 유니크한 한 포트 번호를 서비스에 할당하고, 모든 노드에
그 포트로 들어온 요청을 그 서비스에 속한 팟 들로 라우팅해준다.

신기한 건 어떤 노드에 요청해도 모든 팟에 요청이 간다는 것이다. 그 노드에 속한
팟으로 가는 것도 아니고, 모든 팟에 요청이 분산된다. 내부에서 kube-proxy가 라우팅해준다는
것 같다. 라우팅 정책은 kube-proxy 설정에서 라운드 로빈, 요청 수 기반 등등으로
바꿀 수 있다.
