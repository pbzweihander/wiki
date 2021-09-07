# ALB 인그레스 컨트롤러

> 참고: [쿠버네티스 인그레스](kubernetes-ingress)

[ALB 인그레스 컨트롤러](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)는
AWS의 어플리케이션 로드 밸런서를 이용한 인그레스 구현체다.

## ALB 인그레스 컨트롤러의 모드

ALB 인그레스 컨트롤러에는 IP 모드와 인스턴스 모드라는 두 가지 모드가 있다.
원래 ALB 뒤에 물려있는 대상 그룹에도 두 가지 대상 유형이 있는데, IP를 대상으로
지정할 수도 있고 EC2 인스턴스를 대상으로 지정할 수도 있다. ALB 인그레스
컨트롤러도 대상 그룹의 대상 유형을 지정하여 동작할 수 있게 만들어 놓은 것 같다.

### IP 모드

> 참고: [EKS의 VPC CNI](eks-vpc-cni)

EC2의 내부 IP를 이용해 팟에 직접 트래픽을 던지는 모드이다. 각 팟에 할당된 내부
IP를 사용해서 별다른 오버헤드가 없다는 것이 장점이다.

모든 것이 잘 동작하는 상황에서는 오버헤드도 적고 깔끔한 희망적인 모드인데,
문제점이 좀 있다.

IP 모드에서는 각 팟의 IP가 대상 그룹에 직접 등록되어있는 형태이므로, 팟이 새로
뜨거나 죽으면 대상 그룹에서 새 IP를 등록하거나 기존 IP를 지워주어야한다.
이 일을 ALB 인그레스 컨트롤러가 한다.

그런데 이 동작은 ALB 인그레스 컨트롤러가 서비스 엔드포인트의 변화를 감지하고,
AWS API를 쳐서 대상 그룹을 변경하기를 요청하고, AWS가 요청을 받고 대상 그룹을
수정하는 과정을 거치다보니 매우 느리다. 팟이 하나 죽어서 대상 그룹에서 그 팟이
떨어지기 까지 10초가 걸리는 것도 목격했다.

10초가 걸린다는건, 실제로 요청을 받을 수 없는 IP로 트래픽이 10초동안 라우팅된다는
뜻이다. 10초 동안 일부 유저는 502 Bad Gateway를 보게 된다.

여러 업데이트를 통해 이 과정을 빠르게하고 최적화했지만 근본적으로 딜레이가 있는
동작이라서 한계가 있을 것 같다.

이 문제를 해결하는 방법 중 하나는 팟에 PreStop 훅을 걸어, 팟이 종료될 때 서비스
엔드포인트에서 제거되어도 실제 대상 그룹에서 제거되기 까지 좀 더 기다리는 것이다.

### 인스턴스 모드

> 참고: [쿠버네티스 NodePort 서비스](kubernetes-nodeport-service)

서비스가 NodePort 타입일 때만 쓸 수 있다. 대상 그룹에 클러스터의 모든 인스턴스와
해당 서비스의 포트를 등록한다. 요청이 들어오면 healthy한 아무 노드에 요청이 가지만,
노드 내부에서 kube-proxy가 서비스에 속한 팟 중 하나로 라우팅해준다.

이 모드의 장점은 대상 그룹의 팟 각각의 IP를 등록하지 않아도 된다는 것이다.
항상 모든 노드가 대상 그룹에 등록되어있으니 노드 수 만큼의 요청을 받을 수 있는
대상이 있어서 가용성이 높고, IP를 등록하거나 제거하는 동작이 없어서 IP 모드의
문제점을 겪지 않는다.

팟이 요청을 받을 수 있는지 아닌지 (Ready 상태인지 아닌지) 를 판별하고 트래픽을
라우팅하는건 kube-proxy가 하는 일이고, kube-proxy는 쿠버네티스 클러스터 안에서
팟의 상태를 직접 볼 수 있으므로 ALB 인그레스 컨트롤러가 참조하는 것 보다 훨씬
빠르다. (그럴 것이라 기대한다.)


## 벤치마크

실제로 IP 모드와 인스턴스 모드의 가용성을 HTTP 벤치마크 툴인 [wrk](wrk)으로
비교해보자.

ALB 인그레스 컨트롤러 1.1.5 버전에서 https://github.com/pbzweihander/kubia 를
샘플 HTTP 서버로 삼아 테스트했다.

팟을 3개 띄운 후 초 당 60개의 요청을 보내다가 팟 하나를 죽여보고 에러가 생기는지
테스트해보았다.

**IP 모드일 때**

```bash
$  wrk2 -R60 -t7 -c100 -d10m http://<redacted>.com
Running 10m test @ http://<redacted>.com
  7 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   501.03ms    2.04s   10.05s    95.21%
    Req/Sec     4.53     12.99    82.00     91.24%
  1291 requests in 28.08s, 221.99KB read
  Socket errors: connect 0, read 0, write 0, timeout 286
  Non-2xx or 3xx responses: 20
Requests/sec:     45.98
Transfer/sec:      7.91KB
```

1291개의 요청 중 286개는 타임아웃, 20개의 요청은 502 Bad Gateway가 발생했다.

**인스턴스 모드일 때**

```bash
$ wrk2 -R60 -t7 -c100 -d10m http://<redacted>.com
Running 10m test @ http://<redacted>.com
  7 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    44.40ms    7.66ms  82.56ms   90.82%
    Req/Sec     8.45     20.95    80.00     86.54%
  1274 requests in 20.07s, 216.48KB read
Requests/sec:     63.46
Transfer/sec:     10.78KB
```

1274개의 요청 중에서 에러가 한 건도 발생하지 않았다.

## 결론

위에서 사용한 ALB 인그레스 컨트롤러는 1.1.5 버전으로, 최신 버전에서는 IP를 대상
그룹에 등록하는 속도가 개선되었다고 한다. 그러므로 최신 버전에서는 위와 같은 문제가
발생하지 않을 수도 있다. 그러나 구조적으로 IP 모드는 어쩔 수 없이 오버헤드가 발생하고
인스턴스 모드가 훨씬 튼튼하며 대규모 요청을 잘 받아낼 수 있을 거 같다는 생각이
든다.
