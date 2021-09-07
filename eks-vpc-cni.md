# EKS의 VPC CNI

> 참고: AWS Documentation의 [Pod Networking (CNI)](https://docs.aws.amazon.com/eks/latest/userguide/pod-networking.html)

CNI는 컨테이너 네트워킹 인터페이스의 약자이다. 각 컨테이너가 어떻게 네트워크에
접근하고, 호스트는 컨테이너에 어떻게 네트워크를 할당하고 해제하는지를 정의한
명세이며, https://github.com/containernetworking/cni 에서 볼 수 있다.

AWS EC2 인스턴스는 한 인스턴스에 여러 내부 IP를 지정할 수 있는데, 이를 이용해
CNI를 구현하는 것이 AWS VPC CNI이다. https://github.com/aws/amazon-vpc-cni-k8s 에서
코드를 볼 수 있다.

EC2 인스턴스는 인스턴스 타입 별로 지정해줄 수 있는 내부 IP
갯수가 정해져있는데, 그래서 EKS에서 EC2 인스턴스를 노드로 사용할 경우 노드 당
최대 팟 갯수가 인스턴스 타입에 따라 정해진다.
