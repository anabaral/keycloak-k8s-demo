# PC 환경 kubernetes 에서 keycloak 연습해 보기

이 프로젝트는 다음의 특징을 갖습니다.
- [keycloak-containers-demo](https://github.com/anabaral/vagrant_examples/tree/master/vbox_kube) 프로젝트에서 발전시켰습니다
- [vagrant_examples/vbox_kube](https://github.com/anabaral/keycloak-containers-demo) 프로젝트로 생성한 
  Windows PC + Virtualbox 환경에서 구동됩니다.
- keycloak의 기본 기능을 연습해 보는 의미가 강합니다.
- 따라서 설치를 할 때 무언가 많이 먼저 준비하고 하기 보다는, 일단 단순하게 띄워 보고 조금씩 개선하는 형태로 할 겁니다.
  그래서 연습이 다 끝난 후 실 환경에서 설치하려고 할 때는 오히려 번잡할 수 있습니다.
  이에 대비되는 방식은 이를테면 helm chart 설치를 할 때 일단 chart를 다운로드 받은 후 압축을 풀어 템플리트 파일들을 손본 후 설치하는 식인데
  이게 일반적이고 더 깔끔하지만 공부하는 입장에서는 왜 그걸 하는지 잘 와닿지 않을 수 있습니다.

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
<pre><code>helm repo add codecentric https://codecentric.github.io/helm-charts 
helm install mailhog codecentric/mailhog  </code></pre>

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

## private docker registry 설치

어플리케이션을 띄우고 버전 관리를 하려면 자체 도커 이미지 저장소가 있어야 할 것입니다.
이것도 역시 helm으로 비교적 쉽게 설치됩니다.

<pre><code>$ helm install registry stable/docker-registry </code></pre>

편의상 모든 차트를 네임스페이스 지정 없이 설치하고 있는데, 이렇게 하면 모두 default 네임스페이스에 설치됩니다.

그리고 이렇게 설치한 registry 는 다음과 같은 특징을 가집니다:
- https 접근이 아닌 http 접근을 허용합니다. 
- registry는 브라우저로 접근하는 게 아닙니다. 한편으론 <code>docker build/push</code> 명령으로 접근하고 한편으론 k8s pod 내에서 접근할 것입니다.

이러한 제약 때문에 동일한 URL로 이곳저곳에서 접근이 가능한 방법을 모색해 보았는데 결국 다음으로 정했습니다:
- nodeport 기반으로 서비스를 설정
- 모든 노드의 /etc/hosts 에 호스트명 registry.k8s.com 을 설정
  <pre><code>$ sudo vi /etc/hosts
  ...
  192.128.205.10       ... registry.k8s.com  # nodeport니까 아무 호스트나
  </code></pre>
- 모든 노드에서 docker 가 http 접근이 되도록 설정
  <pre><code>$ sudo vi /etc/docker/daemon.json
  {
      "insecure-registries": ["registry.k8s.com:31000"]
  }
  $ sudo systemctl restart docker
  </code></pre>

※ docker restart 할 때 당연히 지금까지 설치된 pod들이 재시작합니다. 유의해 주세요.
※ 이 저장소는 인증을 요하지 않는 저장소입니다. 실제로는 인증이 필요하겠죠.. 앞으로의 과제로..

## js-console 설치

keycloak-containers-demo 프로젝트에서 SSO 테스트 및 공유정보 확인용으로 사용했던 js-console을 여기서도 설치해 봅시다.
이 이미지를 registry에 등록하고 띄우는 방식을 쓰겠습니다. 이미지는 앞서 프로젝트에서 만든 것을 쓸 수도 있는데 처음부터 한다고 가정해 보겠습니다.

keycloak에서의 설정과 js-console에서의 설정 선후관계가 엇갈리는데 일단 몇 가지 가정을 깔겠습니다:
- http://demo.k8s.com:30080 으로 어플리케이션에 접속할 생각입니다.
- 당연히 PC의 hosts 파일에 <code>192.128.205.10  demo.k8s.com </code> 을 등록해야 합니다. 
- 레지스트리 url은 registry.k8s.com:31000/demo-js-console 으로 할 생각입니다.

Keycloak UI 에서 demo realm 의 clients 메뉴를 선택하고
- js-console 을 추가합니다. 추가 후 
- Settings 탭에서 Valid Redirect URIs = http://demo.k8s.com:30080/* , Web Origins = http://demo.k8s.com:30080 를 입력합니다.
- Installation 탭에서 Format Option 을 'Keycloak OIDC JSON' 으로 택하면 나오는 json 내용을 복사해 둡니다.

<pre><code>$ git clone https://github.com/anabaral/keycloak-containers-demo
$ cd keycloak-containers-demo/js-console
$ vi src/keycloak.json   # 이 내용을 위에 복사해 둔 json으로 바꿉니다.
$ vi index.html          # 여기 script 태그가 있는데 그 src 내용을 https://keycloak.k8s.com:32443/auth/js/keycloak.js 로 바꿉니다.
$ vi build.sh
#!/bin/sh
IMG_URL=registry.k8s.com:31000/default/demo-js-console
docker build -t demo-js-console .
docker tag demo-js-console ${IMG_URL}
docker push ${IMG_URL}
$ sh build.sh
</code></pre>
여기까지 잘 되었으면 좋겠네요. 뭔가 문제가 있다면 하나씩 차근차근 풀어야 합니다.

이제 본격적으로 deploy해 볼 차례입니다. js-console-deploy.yaml 을 작성합니다.
<pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
  generation: 1
  labels:
    app: js-console
    release: js-console
  name: js-console
  namespace: default
spec:
  minReadySeconds: 5
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: js-console
      release: js-console
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: js-console
        release: js-console
    spec:
      containers:
      - image: registry.k8s.com:31000/default/demo-js-console:latest
        imagePullPolicy: Always
        name: js-console
        ports:
        - containerPort: 8000
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      #securityContext:
      #  fsGroup: 1000
      #  runAsUser: 1000
      terminationGracePeriodSeconds: 30
status:
</code></pre>
이제 적용합니다.
<pre><code>kubectl apply -f js-console-deploy.yaml</code></pre>

이번엔 서비스입니다. js-console-svc.yaml 파일을 작성합니다.
<pre><code>apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    app: js-console
  name: js-console
  namespace: default
spec:
  ports:
  - name: http
    port: 8000
    nodePort: 30080
    protocol: TCP
    targetPort: 80
  selector:
    app: js-console
    release: js-console
  type: NodePort
</code></pre>
마찬가지로 적용합니다.
<pre><code>kubectl apply -f js-console-svc.yaml</code></pre>

## openldap 설치 및 설정

openldap을 설치해 봅시다.
<pre><code>$ helm install openldap stable/openldap
You can access the LDAP adminPassword and configPassword using:
  kubectl get secret --namespace default openldap -o jsonpath="{.data.LDAP_ADMIN_PASSWORD}" | base64 --decode; echo
  kubectl get secret --namespace default openldap -o jsonpath="{.data.LDAP_CONFIG_PASSWORD}" | base64 --decode; echo
You can access the LDAP service, from within the cluster (or with kubectl port-forward) with a command like (replace password and domain):
  ldapsearch -x -H ldap://openldap.default.svc.cluster.local:389 -b dc=example,dc=org -D "cn=admin,dc=example,dc=org" -w $LDAP_ADMIN_PASSWORD
</code></pre>
설치 직후에 로그인 패스워드를 어떻게 얻는 지를 메시지로 출력하고 있습니다. 만약 바꾸고 싶다면 위의 설명을 참조해서 적용해야겠죠.

그런데 현재 virtualbox 기반의 호스트에서 openldap.default.svc.cluster.local 식의 호스트 접근이 불가합니다. (왜 그런지 추가조사는 필요합니다)
그래서 CLI 방식으로 ldap을 관리하는 걸 과감히(?) 포기하고 관리용 UI인 phpldapadmin 을 설치하기로 합니다.
<pre><code>$ helm repo add cetic https://cetic.github.io/helm-charts
$ helm install phpldapadmin cetic/phpldapadmin
</code></pre>

설치하면 사용 가이드 메시지가 출력되는데 이는 무시합시다. Ingress 설정으로 풀겠습니다.
- PC의 hosts 에 <code>192.128.205.10  phpldapadmin.k8s.com </code> 을 등록합니다.
- Ingress 를 추가합니다.
  <pre><code>apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
    generation: 1
    name: phpldapadmin
    namespace: default
  spec:
    rules:
    - host: phpldapadmin.k8s.com
      http:
        paths:
        - backend:
            serviceName: phpldapadmin
            servicePort: 80
          pathType: ImplementationSpecific
    tls:
    - hosts:
      - phpldapadmin.k8s.com
      #secretName: keycloak-tls-secret-2020
  status:
    loadBalancer: {}
  </code></pre>

이것만으로 끝이 아닙니다. 단순 설치만 해서는 앞서 설치한 openldap을 가리킬 리가 없으니까요.

다음과 같이 추가로 설정합시다:
<pre><code>$ kubectl get deploy -n default -o yaml > phpldapadmin-deploy.yaml
$ vi phpldapadmin-deploy.yaml # LDAP_HOSTS 설정하자
...
spec:
      containers:
      - envFrom:
        - configMapRef:
            name: phpldapadmin
        env:
        - name: PHPLDAPADMIN_HTTPS
          value: "false"
        - name: PHPLDAPADMIN_LDAP_HOSTS
          value: openldap.default
...
$ kubectl apply -f phpldapadmin-deploy.yaml
</code></pre>

여기까지 하면 https://phpldapadmin.k8s.com:32443/ 으로 접속이 됩니다.
로그인 입력폼에 다음을 입력하여 로그인합니다:
- Login DN: cn=admin,dc=example,dc=org
- Password: 비밀번호( LDAP_ADMIN_PASSWORD 에 해당하는 값 = <code>kubectl get secret --namespace default openldap -o jsonpath="{.data.LDAP_ADMIN_PASSWORD}" | base64 --decode; echo</code> 으로 얻은 값)

필요한 만큼 그룹과 사용자를 추가해 주세요.

그 다음 keycloak에서 이 ldap을 사용하도록 설정합니다:
- Demo realm 에서 User Federation 메뉴를 선택하고, Add provider...ldap 을 선택합니다.
- Connection URL에 위의 메시지에 나온 URL인 ldap://openldap.default.svc.cluster.local:389 을 입력합니다. k8s cluster 내의 통신이니까 이 접근이 됩니다.
  [Test Connection] 으로 확인해 보시구요
- Users DN=dc=example,dc=org , Bind DN=cn=admin,dc=example,dc=org 으로 부여합니다.
  이건 연습이니까 기본적으로 주어진 설정 내에서 사용하려고 값을 준 것인데 ldap을 다르게 활용하신다면 적절히 고쳐 주시면 됩니다.
- Bind Credential 에는 위의 LDAP_ADMIN_PASSWORD 에 해당하는 값을 줍니다. [Test Authentication] 으로 확인해 보세요
- [Save] 하고 [Synchronized all users] 하시면 됩니다.


## Jenkins 설치 및 설정

Jenkins를 설치해 보겠습니다. 이것 역시 chart를 수정해서 설치하는 방식이 아니라 일단 기본구성으로 설치해 보고 조금씩 수정해 나가는 전략을 쓸 겁니다.
<pre><code>$ helm install jenkins stable/jenkins
NAME: jenkins
LAST DEPLOYED: Thu Jun 18 08:40:42 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace default jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=jenkins" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080
  kubectl --namespace default port-forward $POD_NAME 8080:8080

3. Login with the password from step 1 and the username: admin

4. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http:///configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos
For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/</code></pre>
출력되는 메시지를 잘 읽어보면 다음 내용입니다:
- admin 비밀번호를 얻는 방법이 있습니다. 
- port-forward 가 필요하다고 나옵니다. 하지만 이 방식을 쓰지는 않고 nodeport를 쓸 생각입니다. (ingress 추가해서 쓸 수도 있습니다)
- configuration-as-code 방식이 최신버전에 도입됩니다.

이 메시지들과는 별개로 jenkins는 정상부팅되지 않을 겁니다. 기본적으로 설정된 persistent volume claim 이 없는 값이거든요.
이걸 해결하기 위해 다음과 같이 설정합니다 (내용을 단순화했습니다) :
<pre><code>spec:
  template:
    spec:
      hostAliases:
      - hostnames:
        - keycloak.k8s.com
        ip: 192.128.205.10
      volumes:
      - name: jenkins-home
#        persistentVolumeClaim:
#          claimName: jenkins
        hostPath:
          path: /vagrant/keycloak/test2/jenkins_home  # 적절한 디렉터리를 잡습니다
          type: ""
</code></pre>

이렇게 하면 일단 뜹니다.


