# PC 환경 kubernetes 에서 keycloak 연습해 보기

이 프로젝트는 다음의 특징을 갖습니다.
- [keycloak-containers-demo](https://github.com/anabaral/vagrant_examples/tree/master/vbox_kube) 프로젝트에서 발전시켰습니다
- [vagrant_examples/vbox_kube](https://github.com/anabaral/keycloak-containers-demo) 프로젝트로 생성한 
  Windows PC + Virtualbox 환경에서 구동됩니다.
- keycloak의 기본 기능을 연습해 보는 의미가 강합니다.
- 따라서 설치를 할 때 무언가 많이 먼저 준비하고 하기 보다는, 일단 단순하게 띄워 보고 조금씩 개선하는 형태로 할 겁니다.
  그래서 연습이 다 끝난 후 실 환경에서 설치하려고 할 때는 오히려 번잡할 수 있습니다.

## keycloak 설치 및 설정

다음 명령으로 시작합니다:
<pre><code>kubectl create -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/kubernetes-examples/keycloak.yaml</code></pre>

- 설치는 간단히 완료되지만 이것만으로 충분하지는 않은데, 계속 보강할 겁니다.
- keycloak은 위의 설치로 default 네임스페이스에 설치됩니다. 고쳐야 할 것 같은데 여기서는 실습이니 그냥 넘어가겠습니다.

앞서 서술한 가정에 따르면 ingress는 이미 설치되어 있습니다. (혹시 아니라면 아래와 같이 나오도록 설정하세요)
<pre><code>$ kubectl get svc -n ingress-nginx ingress-nginx
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   LoadBalancer   10.101.2.126   <pending>     80:31782/TCP,443:32443/TCP   5d23h</code></pre>

keycloak을 위한 ingress 를 설정합시다.
- PC의 hosts 파일에 내용을 추가합니다.
  <code>192.128.205.10      keycloak.k8s.com </code>
- 다음으로 ingress 를 추가합니다.
  <pre><code>$ KEYCLOAK_HOST=keycloak.k8s.com
  $ cat &gt; keycloak-ingress.yaml &lt;&lt;EOF
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: keycloak
  spec:
    tls:
      - hosts:
        - ${KEYCLOAK_HOST}
    rules:
    - host: ${KEYCLOAK_HOST}
      http:
        paths:
        - backend:
            serviceName: keycloak
            servicePort: 8080
  EOF
  $ kubectl apply -f keycloak-ingress.yaml
  </code></pre>
  
여기까지 하면 위의 ingress-nginx 의 32443 포트로 (https://keycloak.k8s.com:32443) 접속이 가능해집니다. 

### keycloak 에 영속성(persistency)을 부여하자

keycloak은 별도 옵션 없이 설치하면 각종 설정을 자체적으로 띄운 h2db 에 담습니다. 당연히 재시작하면 초기화됩니다.
이를 재시작해도 초기화하지 않으려면 최소한 다음을 해야 합니다.
- PV 연결로 /opt/jboss/keycloak/standalone/data 디렉터리 내용을 보존

다음을 실행합니다.
<pre><code>$ kubectl get deploy -n default keycloak -o yaml &gt; keycloak-data.yaml
$ sed -i -e 's#^      terminationGracePeriodSeconds: 30#&\n      volumes:\n      - hostPath:\n          path: /vagrant/keycloak/data\n          type: ""\n        name: data#' -e 's#^        terminationMessagePolicy: File#&\n        volumeMounts:\n        - mountPath: /opt/jboss/keycloak/standalone/data\n          name: data#' keycloak-data.yaml
$ kubectl apply -f keycloak-data.yaml
</code></pre>

위의 h2db는 나름 괜찮은 db입니다만, 사용자가 늘어나면 mariadb 등 대규모에서의 성능 등이 입증된 DB로 교체해야 할 겁니다.
이걸 교체하려면 조금 설정 양상이 달라질텐데, 요점만 짚고 넘어가겠습니다:
- /opt/jboss/keycloak/standalone/configuration/standalone.xml 에 jboss datasource 설정이 있습니다. 
  여기를 적절히 수정함과 동시에 영속성(persistency)을 부여해야 합니다.

### Keycloak 에 Realm 을 추가하고 설정하자

처음의 keycloak 은 Master realm 뿐입니다. 이 realm은 자체 관리를 위한 것이므로 새로 realm을 추가해 봅시다.
- [Add Realm] 버튼을 누르고 이름을 "demo" 로 부여한 후 [Create] 버튼으로 저장
- 추가한 Realm 에서 Realm Settings 항목을 선택하고 Email 탭을 설정
  - Host: mailhog.default    # 메일 테스트 전용서버. 뒤에 설치할 것인데 설치하고 나면 Test Connection 으로 확인합니다.
  - Port: 1025
  - From: admin@demo-host    # 적당히 부여한 것.

## helm 설치

다음 명령들로 설치하고 기본 저장소도 셋업합니다.
<pre><code>$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/ </code></pre>


## Mailhog 설치

Mailhog 은 메일 테스트 전용 서버입니다. 테스트용이라는 특성을 감안해야 이해할 수 있는 행동을 합니다.
다음 명령으로 설치합니다.
<pre><code> helm repo add codecentric https://codecentric.github.io/helm-charts 
helm install mailhog --set env.service.nodePort.smtp=31025 codecentric/mailhog  </code></pre>

PC의 브라우저에서 접근해야 하니까 PC의 hosts 파일에 <code>${k8s-head 호스트의 IP} mail.k8s.com</code> 문장을 추가합니다. 
(같은 IP로 keycloak.k8s.com 이 등록되어 있을 테니 합치는 게 낫겠네요)
<pre><code>$ vi mailhog-ing.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
  name: mailhog
  namespace: default
spec:
  rules:
  - host: mail.k8s.com
    http:
      paths:
      - backend:
          serviceName: mailhog
          servicePort: 8025
  tls:
  - hosts:
    - mail.k8s.com

$ kubectl apply -f mailhog-ing.yaml
</code></pre>

이렇게 하면 https://mail.k8s.com:32443 으로 접근 가능합니다.




  
