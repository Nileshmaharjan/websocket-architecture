# Infrastructure Refactor Master Roadmap

---

# 서론

적은 수의 개발자로 높은 기술적 난이도의 다양한 마이크로서비스가 굴러가는 아키텍처를 관리하기 위해서는 개인의 역량에 의존하는 것이 아닌 시스템에 의한 관리가 선행되어야 합니다.

현재는 백엔드 개발자가 기능 구현 이외에 인프라 세팅과 운영 이슈에 시간을 빼앗기는 구조입니다. 이 때문에 개발 속도도 저하되고 장기적으로 인프라 비용 증가와 기술 부채로 인한 리스크까지 우려되는 상황입니다.

따라서 앞으로 서비스가 늘어나더라도 흔들리지 않을 단단한 토대를 이번 기회에 마련하고자 합니다.

개발 용어 중 “골든 패스(Golden Path)”라는 말이 있습니다. (A standardized, opinionated, and supported workflow for building software, guiding developers through common tasks like setting up infrastructure or deploying apps, ensuring consistency, speed, and adherence to best practices, often part of an Internal Developer Platform(IDP).

리팩토링이 성공적으로 이루어진다면 테라폼(Terraform)과 깃옵스(GitOps)가 인프라와 배포의 복잡성을 대신 처리해줄 것이며, 로컬에서 도커 컴포즈와 씨름하거나 배포 후에야 테스트가 가능한 기이한 개발 환경에서 벗어날 수 있을 것이고, 복잡한 수동 권한설정(k8s및 gcp에 수동으로 진행하던 것들)에 머리쓸 필요가 없어집니다. 남는 리소르를  서비스 코드의 퀄리티 및 구현속도에만 집중할 수 있도록 세팅해두는 것이 제 목표입니다.

DevOps에 관심이 있다면 구현에 참여해도 되지만, 그렇지 않다고 하면 제가 구축을 해두고, 다른 분들은 이의 동작 과정 및 사용 방법을 이해하고 (사용법을 아는 것만으로는 부족합니다. 반드시 이해하고 넘어가야 합니다) 앞으로의 개발에 활용할 수 있을 정도로 봐주시면 될 것 같습니다.

---

# 설계 원칙 및 개괄

### 1. Immutable Infrastructure

모든 인프라(GCP)와 배포(K8S)는 코드로 정의되며, 콘솔에서의 수동 변경을 금지한다.

Immutable Infrastructure(불변 인프라)는 말 그대로 한 번 생성되면 절대 변하지 않는 인프라를 의미합니다. 쉽게 말해 서버에 들어가서 설정을 바꾸거나 업데이트를 하는 것이 아니라, 변경 사항이 생기면 아예 싹 다 밀어버리고 새로운 서버(컨테이너)로 교체하는 방식입니다.

그런데 Docker와 K8s를 사용했다면, 이미 Immutable Infrastructure를 실천하고 있던 것 아닌지 의문이 들 수 있습니다. 사실 Application Level에서는 이미 Immutable Infrastructure을 실천하고 있던 것이 맞습니다. 하지만 이번 기회에 인프라 레벨, 즉 Cloud Resources까지 포함한 완전한 불변성을 실현하고자 합니다.

기존의 인프라는 반쪽짜리 Immutability의 구현이였습니다. 비유하자면, 건물 내부의 인테리어(App)은 모듈식인데, 건물 자체(Infra)는 시멘트를 덕지덕지 발라놓은 상태임을 의미합니다.

Application의 경우 저희 시스템에서 dev로 푸쉬를 진행하게 되면, 이미지가 업데이트 될 때 기존 컨테이너의 수정이 아닌 새로운 컨테이너로의 교체입니다. 이건 완벽한 Immutable 방식이며, 유지될 예정입니다.

하지만 인프라는 가변(Mutable)이었습니다.

예를 들어볼까요? 

```jsx
gcloud projects add-iam-policy-binding ... --role="roles/pubsub.editor"
```

이러한 명령어를 치거나, 혹은 gcp console의 IAM에서 다음과 같은 권한을 주게 되면, 클라우드 환경의 상태가 영구적으로 변했습니다.

관리만 잘 되면 되는 것 아닌가? 라고 생각할 수도 있습니다. 그래서 실제로 과거(2025년 4~5월정도)의 코드를 보게 되면, readme파일에 실행한 gcp role binding들이나 service account 관련 명령어들이 남아 있습니다.

하지만 이러한 방식으로는 10~20개가 다 되어가는 모든 service에 대한 설정들을 안정적으로 유지보수가 불가능합니다. 누가 이 명령어를 쳤는지, 언제 쳤는지, 왜 쳤는지 기록이 코드에 없습니다(추적은 가능합니다). 

만약 누군가가 실수로 특정 권한을 삭제한다면? 삭제 되었다는 사실도 모르고 동작하지 않는 이유를 kubectl 명령어를 통해 찾는 데에 시간과 노력을 쏟아야 합니다.

만약 dev에는 설정해둔 k8s 설정을 staging에는 누락한 후 dev와 똑같은 설정으로 동일하게 배포했는데 staging에서의 미동작 원인을 디버깅해야한다면? 마찬가지로 kubectl 명령어를 통해 원인을 분석하고 찾는 데에 시간과 노력을 쏟아야 합니다.

결론적으로 클러스터가 오래될수록 Snowflake Server가 됩니다. 너무 섬세해서(좋은 것이 아닙니다) 건드리기 무섭고, 에러가 났을 때, 실수를 했을 때 특히 더 건드리기 무서운 상태가 되는 것입니다.

그렇다면 어떻게 완전한 Immutable Infrastructure를 구축할 수 있을까요? (참고로 자세한 사항들은 후술할 예정이며, 이 섹션에서는 개괄만 담당할 예정입니다.)

Terraform을 도입하고자 합니다. Terraform을 도입한다는 것은 GCP Resource(IAM, VPC, GKE 설정, GCP Buckets)조차도 애플리케이션처럼 다루겠다는 뜻입니다.

### 2. Separation of Concerns

제가 아키텍처에서 가장 중요하다고 생각하는 요소이며, 여러 번 강조한 바 있을 것이라 생각합니다. 실제로도 소프트웨어 설계에 있어서 가장 중요한 원칙 중 하나입니다. 다만 이제는 인프라 및 배포 환경에 대해서도 이를 중시하는 환경을 갖춰보고자 합니다.

현재 저희의 백엔드 마이크로서비스의 레포의 소스코드를 보게 되면, 애플리케이션 저장소 안에 소스 코드, dockerfile, kubernetes helm yaml files, github actions 파일들, 그리고 가끔은 인프라 설정 스크립트(gcloud)까지도 혼재되어 있었습니다. 이는 사실 굉장히 큰 문제입니다.

우선 개발자의 입장에서의 인지부하가 발생합니다. 백엔드 개발자가 서비스 코드와 비즈니스 로직의 구현에 집중해도 모자를 판에, k8s deployment 설정, ingress routing 규칙, 심지어 gcp 권한까지도 신경을 써야 하는 환경입니다.

해당 helm chart들이나 복잡한 workflow 파일들을 이해하지 못하여 다른 작동하는 서비스의 파일을 복사 붙여넣기 해서 사용해서 크게 신경을 쓰지 않았다 해도 이는 문제입니다. 왜냐하면 나중에 설정을 바꾸기 위해서는 모든 레포를 돌아다니며 수정해야만 합니다. 사람은 반드시 실수합니다. 빼먹은 수정사항이 없기를 사람에게 기대할 수는 없습니다. 반면에 컴퓨터는 절대로 실수하지 않습니다. 공용 차트로 지정해둔다면, 수정사항이 전역적으로 적용됩니다.

뿐만 아니라 권한의 모호성까지 야기됩니다. 앱을 배포하는 권한과 인프라를 수정하는 권한은 엄연히 역할이 다릅니다. 역할이 다른데, 한 저장소에 위치시켜서는 안됩니다.

현재 한 백엔드 마이크로서비스의 레포에 있는 복잡한 덩어리를 세 가지 명확한 계층(layer)로 분리할 계획입니다. 즉, 여러분들은 만약 devops시스템 구축에 관심이 없다면, application layer에서 비즈니스 로직을 구현하는 데에만 집중하면 됩니다. (후술할 세 layer 중 layer3에 해당합니다.)

[layer 1] physical infrastructure(물리 인프라 계층) - Terraform

역할: 서비스를 위한 기초 공사, 토대

내용: VPC네트워크, GKE 클러스터, 데이터베이스(Cloud SQL), 스토리지(GCS), IAM 등.

핵심 변경점: VPC 도입

**VPC에 대한 부연설명: VPC(Virtual Private Cloud)는 Google Cloud라는 거대한 땅 위에 짓는 우리의 “사옥”. 전에는 신경을 크게 안썼는데 왜 이제 신경써야하나요? 기존 Autopilot 체계에서는 구글이 사용하는 Default VPC(공용 사무실)을 사용해 별도 설정 없이도 서비스가 잘 굴러갔습니다.

구글 클라우드 프로젝트를 생성하면, 구글이 Default VPC라는 이름으로 이미 지어진 “공용 사무실”을 주고, AutoPilot 사용 시 별 설정을 따로 하지 않으면 해당 공용 사무실 구석에 자리를 잡고 서비스를 띄워줘서 신경을 쓰지 않아도 되었음.

지금까지 Autopilot 쓸 때에는 VPC의 존재조차 몰라도 알아서 잘 됐는데, 왜 굳이 복잡하게 직접 만들어야하는지, 혹시 이제부터 서버 늘리고 줄이기를 (Scale in/out)을 사람이 수동으로 진행해야 하나?

No, 여전히 스케일링은 100% 자동화됩니다. Default VPC는 구글이 임의로 구획을 해놓은 땅이며, Autopilo은 그 위에서 알아서 건물을 부수고 지으며 확장을 해 주었는데, Standard 환경에서는 우리가 이 땅(네트워크)의 크기를 명확히 지정을 해주어야 합니다. Recap: GKE Standard로의 전환의 가장 큰 이유 두 가지는 1) 비용 최적화, 2) 고급 기능 (Telepresence, Logging)

부담이나 로드가 크게 들어가진 않을지? VPC는 매 번 하는 운영 작업이 아니고, 초기에 Terraform 코드로 우리 땅은 이만큼(IP대역) 확보해줘, 라고 선언 (set-and-forget)하면 끝나는 일회성 기초 공사

스케일 업/다운은 여전히 자동 - 트래픽 몰릴 때 수동으로 노드 추가하는 일 없음. Gke Standard를 쓰지만 NAP(Node Auto-Provisioning) 기능을 활성화하면 여전히 자동 설정 가능.

한 줄 요약: 땅만 지정해주면(VPC), 늘어나는건 기계가 한다(NAP)

VPC를 직접 관리하면 나중에 IP대역(땅)이 좁아졌을 때 수동으로 늘려야하는지? Yes, 이건 코드를 수정해야 함 - Terraform 코드를 수정해야 함. 하지만, 그럴 일이 없도록 설계하면 됨. 클라우드 내부망 주소 (Privae IP)는 무료 자원이므로, 처음에 초대형 대역폭을 확보하고 시작할 수 있음. 예를 들어 서브넷 마스크를 /16으로 지정하면 사용 할 수 있는 IP는 2^16=65536, 마이크로서비스 하나는 파드 5개 정도를 사용, 서비스 100개가 되어도 IP는 500개 부근.

[layer 2] Logical Deployment (논리적 배포 계층) - GitOps (ArgoCD)
역할: 경기 규칙 설정, 선수 배치(Configuration)

내용: Replica 개수, 환경변수(Env), 이미지 태그, 도메인 연결 설정(Ingress), Secret 관리 등

핵심 변경점: gitops-manifests 별도 저장소 운영, Push 방식(Github Actions 직접 배포) → Pull 방식(ArgoCD 도입), 소스 코드에서 모든 IP 주소와 비밀번호 제거(No hardcoding, Use External Secrets)

지금까지는 애플리케이션 저장소 안에 helm이라는 폴더를 두고 그 안에 helm chart 공용 파일들을 두고, 개별의 values.yaml, deployment.yaml을 두었습니다. 즉, 애플리케이션 레이어의 코드의 수정이 아닌 환경변수만을 수정하려 할 때에도 전체 CI 파이프라인(빌드-이미지생성)을 다시 돌려야 하거나, 실수로 코드와 설정이 뒤섞여 롤백이 어려운 경우가 존재했습니다. 이제는 정석적인 구조로 앱이 “무엇”을 할지(Code), 그리고 앱이 “어떻게” 실행될지(Configuration)을 물리적으로 찢어놓게 됩니다.

레포지토리가 늘어나서 관리하기 귀찮지 않나요? 초기 설정 때에는 낯설 수 있지만 실제로는 더 편해집니다. 애플리케이션 레이어를 담당하는 개발자는 이 저장소를 수정할 일이 없습니다. 환경 변수 변경이나 리소스 조정이 필요할 때에만 이 저장소를 들여다보면 되며, 기본적으로 제가 관리합니다.

배포 속도가 2-hop이 되며 느려지지는 않는지? 단순 설정 변경(메모리 증설 등) 시에는 무거운 Docker 빌드 과정을 건너뛰고 설정 파일만 수정하면 ArgoCD에 의해 반영되므로 단순 설정 변경 배포가 용이해 짐.

**ArgoCD에 대한 간략한 설명(자세한 설명은 후술)

지금까지 우리는 Github Actions가 클러스터에 접속해서 helm upgrade 명령을 날리는 “Push 방식”을 사용했습니다. 잘 동작했지만, 보안, 그리고 특히 관리의 측면에서 한계가 있었기 때문에 ArgoCD를 도입합니다.

ArgoCD는 24시간 내내 Git의 내용을 바라보다가, Git에 적힌 내용과 실제 상태가 달라지면, 즉시 클러스터를 수정하여 Git과 동일하게 설정해줍니다. (Synchronization)

기존에 잘 되던거 아닌가? 왜 바꿔야하지? 기존에도 Github Actions로 helm upgrade를 실행해 배포했고, 실패하면 이전 커밋으로 되돌려 롤백했고, GCP Console에서 pod 상태를 볼 수 있었는데? 라는 의문에 대한 답변

1. 보안 강화(CI에게 집 열쇠를 맡기지 않는다)
    1. AS-IS(Push): helm upgrade를 실행하려면 Github Actions라는 외부 Saas에게 우리 클러스터의 관리자 인증 정보(KubeCOnfig)를 등록해 두어야 함. GitHub 계정이 해킹되면, 우리 클러스터 전체가 위험해짐
    2. TO-BE(Pull): ArgoCD 도입 후, GithubActions는 슬프게도 클러스터 접속 권한을 박탈당함. CI는 Git에 텍스트 파일(Image Tag)만 수정하고, 실제 배포는 클러스터 내부의 ArgoCD가 안에서 수행함. 즉, 외부인에게 집 열쇠를 맡기지 않음.
2. 롤백(빌드 파이프라인과의 의존성 제거)
    1. AS-IS: 롤백을 하려면 Git Revert 후에 CI 파이프랑니(환경 세팅→빌드→배포)가 처음부터 끝까지 다시 성공하기를 기다려야 했음. 
    2. TO-BE: ArgoCD는 이미 빌드된 이미지로 설정만 교체함. CI 파이프라인 거치지 않고 이전 Revision으로 즉시 되돌림.
3. 가시성(Visibility)
    1. AS-IS: GCP Console은 "파드가 살아있다(Running)"는 물리적 상태만 보여줍니다. 누군가 실수로 운영 환경 설정을 바꿨을 때, 파드가 죽지만 않으면 GCP Console은 "정상"이라고 표시합니다.
    2. TO-BE: ArgoCD는 정합성을 봅니다. "지금 돌아가는 파드의 설정이 Git에 적힌 의도(Intent)와 다릅니다(OutOfSync)"라고 경고해줍니다. 이는 단순한 헬스 체크를 넘어, 설정 오류로 인한 잠재적 사고를 예방해줍니다. (예시: 살아있어도 설정이 다르면 경고)

**Secret & Config Management

ArgoCD의 도입과 함께, 기존에도 문제였던 Secret(IP주소, password등)의 소스 코드 하드코딩을 전수 조사하여 제거합니다.

환경 설정(Config)의 분리

기존에는 소스 코드 안에 (정확히는 helm 폴더 안의 values파일에) redis주소 = x.x.x.x, dp ip = x.x.x.x, password = xxxx 처럼 주소를 하드코딩해 두었습니다. 기존의 빠른 개발을 위해서는 이해가 되나, production 환경에서, enterprise레벨의 개발을 위해서는 그래서는 안됩니다. Application Layer 개발자는 변수명만 알고 있으면 됩니다.

Application layer developer: 변수명만 알고 있으면 됩니다. REDIS_HOST라는 것을 사용하되, 그 값이 무엇인지 알거나, 기재할 필요가 없습니다.

GitOps Repo Layer: gitops-manifests/dev/values.yaml 파일에 REDIS_HOST: redis-master.internal 등으로 정의를 해둡니다.

즉, Redis 주소가 변경되면? Application Layer 개발자는 코드를 건드릴 필요가 없습니다. 인프라 담당자가 GitOps 레포의 텍스트만 수정하면 ArgoCD가 알아서 실행 중인 파드의 Environment Variable을 갈아끼워 줍니다. Re-build조차도 필요 없습니다.

Secret의 분리

그렇다면 redis password, db password 등의 값은? 이것도 GitOps 저장소의 values.yaml에 평문으로 작성하나요?

비밀번호는 Git에도 기재하지 않습니다. 비밀번호는 GCP의 GCP Secret Manager에 저장합니다. 클러스터에 ESO(External Secrets Operator)라는 도구가 배포 시점에 Secret Manger에서 비밀번호를 가져와 앱에 Injection 해주는 방식입니다.

Application Layer 개발 시에는 그저 os.getenv(”DB_PASSWORD”)를 쓰면 되고, 그 값이 어디에서 오는지, 그 값이 오는지는 알 필요가 없습니다. 신경 쓸 필요가 없습니다. 만약 오류가 난다면, 오류가 난다는 사실을 notify만 해주시면, 그 연결 이슈를 해결하는 것은 devops의 역할입니다.

[layer 3] Application Layer (애플리케이션 계층) - Source Code

역할: 실제 비즈니스 로직의 구현

내용: Python, Go, Java 소스 코드, Dockerfile, Unit Test

핵심 변경점: Golden Path(공통 라이브러리 차트) 도입 및 Repository 경량화

신입 저년차 개발자가 우리 마이크로서비스의 레포지토리를 보는 순간 기절할 확률이 높습니다. 인프라 스크립트와 특히 k8s helm 관련 코드들은 처음에는 이해하기 매우 어렵습니다. 백엔드 로직을 짜다가 배포가 터지면 helm chart들을 살펴보아야하는데, learning curve가 높은 편이기 때문에 스스로 파악하기 어렵습니다.

이제 레포지토리에는 복잡한 인프라 스크립트와 K8s manifest 파일들을 모두 삭제합니다. 오로지 서비스 코드, 그리고 Dockerfile만 남습니다.

기존의 거대한 /helm 폴더는 제거하고,

serviceName: some-service
image:
  tag: "v1.0.0"
infra:
  use_db: true
  public_access: true

와 같이 20줄 미만의 설정 파일만있으면, 나머지 injection은 layer 1과 2가 담당합니다.

그렇다면 앞으로의 배포를 위해 K8s 설정을 몰라도 되나요? 네, 몰라도 배포할 수 있도록 만드는 것이 목표입니다. 사내 표준 (보안 설정, HPA, logging 정책 등)을 모두 담아둔 공통 helm chart를 제가 작업하여 제공할 예정입니다. 

나만의 커스텀 설정이 필요하다면? 공통 차트는 강제가 아니라 default를 편리하게 제공하는 것이기 때문에, 예를 들어 커스텀 헤더나 특정 파드에만 GPU 할당이 필요하거나, 사이드카 컨테이너 주입이 필요한 경우(예시: messaging-orchestrator) 이를 위한 공통 차트를 제공하거나, 아니면 override를 허용하도록 유연하게 설계하면 됩니다.

모든 서비스가 하나의 차트를 사용하면, 차트 업데이트 했을 때 오류가 있으면 모든 서비스가 다 다운되는 것 아닌지? 차트 버저닝을 관리하여, 예를 들어 기능이 추가된 v2.0이 릴리즈 되더라도 명시적으로 서비스의 코드를 수정하기 전까지는 v1.0을 사용하게 됩니다. 검증 된 이후 원하는 시점에 버전 업을 진행하면 되므로 blast radius(장애 전파)는 없습니다.

### 3. Developer Experience

MSA 환경에서 백엔드를 개발할 때에 가장 큰 Pain Point는 내 서비스 하나를 띄우기 위해 DB, Redis, 타 마이크로서비스까지 로컬에 다 띄워야한다는 점입니다. 알고 계셨겠지만, 저도 초기에는 매우 무거운 docker-compose로 일일히 다 띄워 테스트하였지만, 점점 규모가 커지며 아예 포기를 한 뒤, dev 서버에 배포한 뒤에 테스트하는 기이한 방식으로 작업을 해왔습니다.

전에도 공유한 적이 있지만, Telepresence라는 도구가 있습니다. Telepresence에 대해서는 뒤에 자세하게 언급하겠지만, 로컬 컴퓨터와 GKE 클러스터를 네트워크로 직접 연결하는 기술입니다. 즉, 내 로컬 코드가 클러스터 내부의 DB, redis 등과, 혹은 타 마이크로서비스와 실시간으로 소통하는 쾌적한 개발 환경을 제공하는 것입니다.

다만 세팅의 난이도도 높으며, GKE Autopilot 상에서는 구축이 불가능합니다. 따라서 새로운 인프라 환경에서는 Autopilot을 사용하지 않는 GKE Standard 환경을 구축하고자 합니다. 비용 절감의 문제도 있지만 (그동안 편의성을 이유로 규모에 비해 과다한 인프라 비용을 지불해왔습니다), Standard 환경에서는 이와 같은 생산성 도구들을 더욱 자유롭게 사용할 수 있는 세팅이 가능합니다.

TO-BE: Telepresence를 활용한 하이브리드 개발

- 지금 개발 중인 example-service라는 서비스 단 하나만 로컬 IDE에서 실행하고, 의존성이 있는 여러 개의 서비스와 DB, Redis 등은 GKE 클러스터에 이미 떠 있는 자원을 그대로 갖다 씁니다.
- 로컬 코드에서 http://another-service를 호출하면, 마치 내 컴퓨터에 있는 것처럼 클러스터 내부의 another-service로 요청이 전달됩니다. Cloud SQL (Dev DB)db에도 별도의 프록시 설정 없이 바로 붙을 수 있습니다.
- 클러스터로 들어오는 실제 API 요청 중, 특정 헤더가 있는 요청만 낚아채서(Intercept) 내 로컬 컴퓨터로 가져올 수 있습니다. 배포하지 않고도 실제 환경에서의 동작을 내 IDE 및 디버거로 찍어볼 수 있습니다.

**Autopilot 환경에서는 이러한 네트워크 터널링 도구들이 요구하는 권한(Privileged Access, NET_ADMIN 등)을 엄격하게 제한하므로 구축이 어렵습니다.

즉, 여러분들은 무거운 인프라는 클라우드에게 맡기고 맥북으로 가볍게 핵심 로직 구현에만 집중할 수 있는 환경을 구축하는 것이 목표입니다.

---

# 1. 거버넌스 및 저장소 전략 (Governance & Repositories)

목표: 시스템의 뼈대가 되는 물리적인 저장소와 논리적인 규칙, 그리고 명명 규칙을 확립

### 1.1. Repository Structure (3-Repo Strategy)

우리는 관심사의 분리(Separation of Concerns) 원칙에 따라 시스템을 물리 인프라(Terraform), 논리 배포(GitOps), 애플리케이션(App)의 3가지 계층으로 나누어 저장소를 운영합니다.

각 저장소는 역할과 권한이 엄격하게 분리됩니다.

| 저장소명 | 관리 주체 | 기술 스택 | 역할 및 포함 내용 |
| --- | --- | --- | --- |
| infra-terraform-main | DevOps | Terraform | [Physical Layer]
GCP 리소스 프로비저닝
- VPC, GKE, IAM, GCS, CloudSQL, Artifact Registry |
| infra-gitops-manifests | DevOps & CI Bot | ArgoCD, Helm | [Logical Layer]
K8s 리소스 상태 정의
- Helm Values, Ingress, K8s SA, SecretStore, Middleware |
| service-[name] | Backend/Frontend | Python, Go, Java, typescript, javascript, Docker | [Application Layer]
비즈니스 로직의 구현
- Source Code, Dockerfile, Unit Test, Migration SQL |

**infra-terraform-main (물리 인프라)**

이 저장소는 변하지 않는 기초 공사를 담당합니다. 서비스 배포와 무관하게, 클라우드 환경 자체를 정의합니다.

- State 관리 (협업을 위한 설정)
    - Backend: GCS Bucket (terraform-state-bucket)
    - 설명: 테라폼이 만든 인프라 현황판(State)를 GCS에 저장하여 공유합니다.
    - Locking: 누군가 인프라를 수정 중일 때, 다른 사람이 동시에 건드리지 못하도록 GCS가 자동으로 Lock을 걸어 사고를 방지합니다. (참고: AWS는 DynamoDB가 필요한 것과 달리 GCP는 GCS Bucket이 자체적으로 Locking을 지원함)

Directory Structure Spec:

```markdown
infra-terraform-main/
├── modules/                      # [공통 모듈] 재사용 가능한 리소스 정의
│   ├── vpc/                      # 사옥 기초 (네트워크, 방화벽)
│   ├── gke/                      # 서버 운동장 (클러스터 설정)
│   ├── database/                 # DB 인스턴스 및 유저 생성 로직
│   └── workload-identity/        # 권한 관리 모듈
├── environments/                 # [환경별 상태] 실제 배포되는 곳
│   ├── dev/
│   │   ├── main.tf               # 모듈을 조립하여 Dev 환경 구성
│   │   ├── backend.tf            # "인프라 장부(State)는 GCS 버킷에 저장해라" 설정
│   │   └── variables.tf
│   └── prod/
└── README.md
```

**infra-gitops-manifests (논리 배포 & 설정)**

이 저장소는 클러스터의 ****현재 상태(Source of Truth)입니다. ArgoCD는 유일하게 이 저장소만을 바라보고 동기화합니다. 또한, CI/CD 파이프라인의 진짜 로직 Template이 이 곳에 저장됩니다.

- 포함 금지 사항: 소스 코드, 빌드 스크립트, 평문 비밀번호(Secrets)

Directory Structure Spec:

```markdown
infra-gitops-manifests/
├── .github/
│   └── workflows/                # [Reusable Workflows] 중앙 관리형 CI 로직
│       ├── ci-java-template.yaml # "빌드하고 태그 업데이트해라" (로직 원본)
│       └── ci-python-template.yaml
├── charts/                       # [Golden Path] 사내 표준 라이브러리 차트
│   └── draesthe-service-chart/   # 모든 MSA가 공유하는 템플릿 (Deployment 등)
├── apps/                         # [App of Apps] ArgoCD 관리용 최상위 그룹
│   ├── dev.yaml                  # "Dev 환경의 모든 앱을 관리해라"
│   └── prod.yaml
├── environments/                 # [실제 배포 값] 환경별/서비스별 설정
│   ├── dev/
│   │   ├── services/             # 비즈니스 앱들
│   │       ├── crmcore/          # crm-core 서비스
│   │       │   └── values.yaml   # "이미지는 v1.0이고, DB 연결해줘" (설정값만 존재)
│   │       └── users/
│   └── prod/
└── README.md
```

**service-[name] (애플리케이션)**

앞으로 개발자 여러분들이 가장 많이 다루게 될 저장소입니다. 인프라 코드와 복잡한 CI 스크립트를 제거하고 순수 코드만 남겨 비즈니스 로직의 구현에 집중합니다.

- Naming Rule(상세한 설명 후술): service-crmcore (접두어 service-로 통일)
- 포함 금지 사항: k8s/ 폴더, helm/ 폴더, IP 주소 및 비밀번호 하드코딩

CICD 전략 (Reusable Workflow)

- 단일 파일 전략: dev.yaml, staging.yaml로 나누지 않고, ci.yaml 하나로 통합합니다. (GitOps의 핵심 철학: build once, deploy anywhere)
- 껍데기 파일: 이 repo의 Workflow 파일은 직접 로직을 수행하지 않고, infra-gitops-manifests에 있는 중앙 템플릿을 호출만 합니다.

Directory Structure Spec:

```markdown
service-crmcore/
├── src/                          # 소스 코드 (Python/Go/Java)
├── Dockerfile                    # 이 코드를 어떻게 실행할지 정의
├── .github/
│   └── workflows/
│       └── ci.yaml               # [Caller] 중앙 템플릿을 호출하는 ~10줄 파일
└── README.md
```

ci.yaml 예시 (L3 Repository 위치):

```yaml
name: CI Pipeline
on:
  push:
    branches: ["main"]
jobs:
  call-central-workflow:
    # L2에 있는 '진짜 로직'을 가져와서 실행함.
    uses: gaia-corporation/infra-gitops-manifests/.github/workflows/ci-python-template.yaml@v1
    with:
      image_name: "crm-core"
    secrets: inherit
```

**GitOps 및 Image Promotion 전략**

Build Once: 이미지는 딱 한 번만 굽는다. (Dev용, Staging용, Production 용 별개 X)

Deploy Selectively: 만들어진 그 이미지를 Dev에는 자동으로 뿌리고, Prod에는 원할 때 뿌림.

Workflow:

- Step 1: 개발자가 코드 Push (자동)
    - 개발자: (기존과는 다르게 main을 이 source로 한다고 가정) main 브랜치에 코드 푸시함 (git push)
    - CI 봇:
        - 이미지 빌드
        - 봇이 infra-gitops-manifests 레포에 가서 dev/values.yaml 파일을 알아서 수정하고 commit & push
        - ArgoCD가 어? Git이 변했네? 하고 Dev 서버를 업데이트 함
- Step 2: Prod 배포 (승격 방식)
    - CI 봇:
        - Dev 배포를 성공시킨 후, 자동으로 Dev(v1.0.0) → Staging 승격 요청이라는 제목의 PR을 하나 띄워줌
    - 개발자
        - PR을 보고 변경 사항(이미지 버전)을 확인함
        - Merge 버튼 딸깍 → ArgoCD가 배포

**ArgoCD Image Updater의 경우 통제 불안성 및 기록 미비로 인해 사용하지 않는 것으로 계획되어 있는 상태임.

### 1.2. Naming Convention (Standardization)

자동화와 식별을 위해 아래 규칙을 엄격하게 준수합니다.

**공통 변수 정의**

이름을 조립할 때 사용할 블록들을 먼저 정의함

| 변수명 | Code | 설명 및 이유 |
| --- | --- | --- |
| Project Prefix | dra | Dr.Aesthetics의 약어.
전역 유일성이 필요한 리소스 (GCP Bucket 등)의 충돌 방지용 접두어 |
| Environment | d, s, p | (참고: 현재 계획상으로는 staging을 deprecate하고 dev와 production을 둘 계획임)
각각 dev, staging, production의 약어.
GCP Service Account 길이제한(30자) 때문에 한 글자로 제한함. |
| Region | seo | asia-northeast3 (Seoul)의 약어
리소스 이름에 지역을 명시하는 경우 사용
예시) 서울 리전에 위치한 개발 환경용 서브넷: sb-d-seo |

**Service Identity & Naming Rules**

모든 애플리케이션(Layer 3)은 식별을 위해서 Full Name(공식 명칭)과 Short Name(식별자)이라는 두 가지 식별자를 반드시 가집니다. 해당 부분의 정의를 정독하여야 뒤따르는 규칙들을 이해할 수 있습니다.

**Full Name (공식 명칭)**

- 용도: 가독성이 필요한 곳 (Github Repository 명, K8s Deployment 명, ArgoCD App Name 등)
- 형식: kebab-case
- 구조
    - 형식: {scope}-{topic}-{function}
    - 설명: 누가-무엇을-하는지
- 제약 사항
    - 기술 스택 기재 금지: 기존처럼 뒤에 -fastapi, -next, -spring, -gin을 붙이지 않습니다.
    - snake case(_)의 혼용을 금지합니다
    - No versioning: 레포지토리의 이름에 new, v2를 절대 넣지 않습니다
    - 모호한 이름 금지: 더 좁은 역할범위를 가짐에도 common, util, manager 등 의미가 불분명한 단어 사용을 지양합니다 (정말로 공용 라이브러리인 경우 common-lib등을 사용하더라도, 의미 범위가 좁은데에도 사용하는 것을 지양하라는 것)

**Naming Components 정의**

- Scope (최상위 분류)
    - Whitelist 방식을 택하여 하단 표의 접두어만 허용합니다.
    - 새로운 Scope 추가 시 승인이 필요합니다.
        - 예시: dermacode의 “치과” 버전이 새로 출시가 된다면, 새로운 Scope인 dentalcode(예시)가 추가됨. 치과 특화 도메인은 해당 Scope의 영역, 치과 특화 ai model도 해당 Scope의 영역, 공통 ai component는 gaia에 위치

| Scope (접두어) | 대상 프로덕트 | 설명 및 결정 사유 | 비고 |
| --- | --- | --- | --- |
| gaia | Common Core | 모든 프로덕트가 공유하는 백엔드/인프라 | 비즈니스 로직이 없는(지향) 순수 기술 기반: user, auth, infra, msg, ai-serving 등 |
| dermacode | Skin AI Analysis | AI 피부 분석/진단/상담 프로그램 | 피부 도메인 특화 로직 |
| patient | Patient App | 환자용 어플리케이션 |  |
| staff | Admin App | 병원 관리자용 앱 |  |
| crm | New CRM | 새로 만드는 v2를 crm이라는 정식명칭의 주인으로 함 |  |
| legacy | Old CRM | 기존 v1 crm 관련을 legacy scope로 격리, 혹은 추가로 deprecated 된 것들 | 구버전 격리용 |
| commerce | Skincare E-commerce | 출시할 스킨케어 이커머스 관련 |  |
| portal | Clinic Web | 병원 공식 홈페이지 |  |
| finance | 없음 | - | 전사 공통 금융 도메인 |

- Topic (업무 영역)
    - 정의: 해당 애플리케이션이 다루는 비즈니스 주제를 의미합니다.
    - 형식: kebab-case, Full English Word
    - 구조: 단수형 명사 1단어를 기본으로 하되, 구체적인 설명이 필수적인 경우 2단어(noun-noun)까지 허용합니다.
    - 원칙
        - Singular Noun (단수형 사용): 집합을 의미하더라도 이름은 단수형으로 통일합니다.
            - (X) users, payments, orders
            - (O) user, payment, order
        - No Abbreviation (약어 금지): Full Name은 가독성이 우선이므로, 임의로 줄여 쓰지 않습니다.
            - (X) treat, rsv, msg
            - (O) treatment, reservation, messaging
        - One Concept (동의어 통일): 같은 개념에는 반드시 하나의 단어만 사용합니다.
            - 예: messaging, sending, communication 등 동일 의미의 개념어 중 messaging을 사용하기로 하였으면 앞으로 통일
    - Naming Tip (헷갈릴 때의 결정 방식)
        - 이 애플리케이션이 메인 대시보드라면? (특히 FE)
            - 예를 들어 aesthe-crm-next처럼 모든 기능을 포괄하는 단 하나의 메인 웹사이트라면, topic은 “main”임.
            - 즉, crm-main-web이 됨. (다른 항목들에 대해서는 후술 내용 참고)
        - 핵심 DB Table, Schema 이름 확인
            - 예를 들어 메인으로 다루는 테이블 명이 transactions라면 Topic은 transaction 입니다.
        - 1단어로 표현 가능여부
            - Treatment 처럼 1단어로 명확하면 그대로 사용
            - Skin이라고만 하면 피부 분석이라는 의미가 퇴색되는 것 같다면 skin-analysis처럼 word-word 구조 사용함
    - 권장 Topic 예시
        - user
        - payment
        - treatment
        - inventory
        - skin-analysis
        - access-control
    - **중요 주의사항: 해당 Topic은 “db”의 logical 분리 대상이기도 합니다. 즉, Scope가 다르더라도, topic 명이 다르도록 설계하는 것이 매우 중요합니다. → 만약 이에 실패한다면 db name에 conflict가 발생하게 됩니다.

- Function (애플리케이션 형태)
    - 해당 코드가 수행하는 기술적인 역할과 형태를 정의합니다.
    - Whitelist: 혼선을 막기 위해 Whitelist에 정의된 표준 접미어만 사용합니다. (정리: Topic 만이 customization의 영역이고, scope와 function은 사전 정의된 것 중에서 사용)
        - 지금까지 구현한 대부분의 백엔드 repo는 api에 해당하고, 대부분의 프런트엔드 repo는 web, app에 해당합니다.
        - lib 부연설명: 혼자 실행(배포)되지 않고, 다른 프로젝트에 설치(import)되어 사용되는 코드. Storybook도 여기에 해당됨.
        - 예를 들어 백엔드의 공통 유틸리티들을 묶은 모듈을 만들거나, dermacode를 쉽게 쓰도록 SDK(API Client)를 배포한다면 그것도 lib에 포함됨. 또한, TypeScript Type이나 Protobuf 파일도 lib에 포함됨. 각각의 이름은 예를 들어 gaia-common-utils-lib, dermacode-sdk-lib, gaia-api-schema-lib 이런식으로 지어지게 될 것임.
        - sidecar 패턴은? 이 레포의 핵심 주인(main container)가 누구인지를 보고, (대부분의 경우) 주인이 API 서버라면 옆에 사이드가가 네 다섯개가 붙어도 Function은 “api”임.

| Function | 설명 | 언제 사용하는가? (결정 기준) |
| --- | --- | --- |
| api | Backend API Server | HTTPC/gRPC 요청을 처리하는 서버가 메인일 때 (BE기본) |
| web | Web Frontend | 브라우저에서 실행되는 웹 프로젝트(React, Nextjs 등, FE기본) |
| app | Mobile App | 앱스토어에 배포되는 모바일 앱 프로젝트(Flutter 등) |
| worker | Standalone Worker | API 서버와 코드가 분리된 독립적인 백그라운드 프로세스일 때만 사용. (즉, API와 코드가 같다면 레포지토리를 따로 만들지 않음) |
| lib | Shared Library | 여러 프로젝트에서 import해서 쓰는 공용 SDK/Package |

**Short Name(식별 코드)**

- 용도: 길이 제한이 엄격한 리소스 (예: GCP Service Account, Database User, GCS Bucket Suffix 등)에 사용합니다.
- 가독성보다 고유성이 중요합니다. 읽기 조금 힘들어도 짧고 유일한 것이 중요합니다. 즉, Short Name만 보고 Full Name이 유추가 안될 수준이라도 문제 없습니다.
- 형식: lowercase (특수문자 제외, kebab 아님)
- 조합 규칙: Fullname을 압축하여 최대 10자 이내로 생성
- 압축 방법:
    - Scope: 표에 정리
    - Topic: Scope와 Function의 축약 후 길이에 따라 최대 10자 제한을 준수하며 자유롭게 설정합니다. Full Name을 정하는 시기(Github Repo 생성)에 동시에 Short Name도 정하여 Readme 혹은 지정된 양식, 채널(미정)에 기재하여야만 합니다. 예를 들어 Scope가 3글자, Function이 3글자로 축약되었다면, 4자 이내로 축약형을 제공하면 됩니다.
    - Function: 표에 정리
    - 예시
        - crm-crm-core-api → crmcoreapi
        - gaia-transaction-ledger-api → galedgeapi (예를 들어 galdgrapi 등으로 가운데 topic 영역은 자유롭게 설정하면 됨, 대신 이후에 변경이 불가능함)
        - crm-main-web → crmmainweb

| Scope (접두어) | 압축 이후 |
| --- | --- |
| gaia | ga |
| dermacode | dc |
| patient | pa |
| staff | sa |
| crm | crm |
| legacy | lgc |
| commerce | com |
| portal | prt |
| finance | fn |

| Function | 압축 이후 |
| --- | --- |
| api | api |
| web | web |
| app | app |
| worker | wrk |
| lib | lib |

**Building Block들의 조립**

지금까지 Full Name, Short Name 그리고, 공통 변수들을 정의하였습니다. 사전에 명확하게 정의된 이 변수들을 활용하여 앞으로 다음과 같은 곳들에서 사용할 변수명들을 명확하고 예측 가능하게 짓게 됩니다:

- Github Repository
- GCP Service Account
- GCS Bucket
- Cloud SQL Instance
- Logical Database
- Database User
- K8s Namespace
- GKE Cluster
- Artifact Registry (Docker 이미지 저장소)
- GCP Secret Manager (비밀번호 저장소)
- Kubernetes Service Account (KSA)
- VPC & Subnet (네트워크)

**기본 변수 및 식별자**

앞서 언급한 기본 변수 및 식별자를 다시 짚고 넘어갑니다

- 기본 변수
    - {Prefix}: dra
    - {Env}: d/p
    - {Region}: seo
- 식별자
    - {Full Name}: 예시 - gaia-user-api
        - {Scope}: 예시 - gaia
        - {Topic}: 예시 - user
        - {Function}: 예시 - api
    - {Short Name}: 예시 - gauserapi

**애플리케이션 리소스 (Layer 3 Dependent)**
애플리케이션과 1:1로 매핑되는 리소스들입니다. 기존에 언급한 것과 같이 저장소의 이름에 service- 접두어가 붙게 됩니다.

| 리소스
(Resource) | 조합 규칙
(Format) | 예시
(Example) | 비고 |
| --- | --- | --- | --- |
| Github Repository | service-{Full Name},
pkg-{Full Name} | service-gaia-user-api,
pkg-crm-statistics-lib | 배포되는 앱의 경우 service- 접두사 뒤에 Full Name을, 설치되는 패키지 혹은 공통 라이브러리 등은 pkg- 접두사 뒤에 Full Name을 배치합니다.
인프라 및 설정 관련은 infra-terraform-main, infra-gitops-manifests 에서 담당합니다. (고정된 이름의 단일 레포지토리) |
| K8s Namespace | {scope} | gaia, finance, dermacode, crm, legacy, commerce, portal, staff, patient | Full Name 중 {Scope}가 namespace의 단위가 됩니다.
 |
| K8s Service Account (KSA) | {Full Name} | gaia-user-api | KSA는 특별한 길이제한 없음
KSA는 GSA와 다르게 -{Env}를 붙이지 않음. GSA의 경우 프로젝트에 대해 유일한 naming이여야하지만, KSA는 cluster 단위로 격리되기 때문에 오히려 다른 cluster에 대해 같은 이름을 갖도록 하는 것이 직관적임 |
| GCP Service Account (GSA) | {Short Name}-{Env} | gauserapi-d
gauserapi-p | GSA의 길이제한인 6~30자를 널널하게 충족함 |
| Secret Manager | {Full Name}-{Env} | gaia-user-api-d
gaia-user-api-p | Secret Manager는 Project 내에서(dr-aesthetics) 유일해야하므로 Env가 배치되어야 함(GSA도 마찬가지)
**매우 중요: GCP에서는 이름이 다르지만 K8s 내부로 가져올 때에는 secret 이름을 gaia-user-api로 통일해서 가져와야 application code가 환경을 신경쓰지 않고 돌아감 |
| K8s Secret | {Full Name} | gaia-user-api | 하단 설명 참고 |
| Logical Database | {Topic} | ledger, user | 해당 사항에 대한 설명이 길어져 하단의 설명을 참고할 것
Cloud SQL Instance의 Naming Convention은 하단의 데이터 및 리소스 표 확인할 것 |
| Database User | {Short Name} | fnledgwrk
(finance ledger worker) | 1 App = 1 DB User, 각 마이크로서비스는 본인에게 할당된 Logical DB로의 접근 권한만을 갖습니다 (Grant) |

**Secret Manager의 데이터구조:

Secret을 그동안 helm values파일에 하드코딩 할 때에, 항목별로 분리해 저장했었습니다. 예를 들어 DB_HOST: xx.xx.xxx, DB_SECRET: xxxxxxxx와 같은 방식으로. 하지만 서비스와 Secret의 1대1 대응을 위하여 json payload로 구조를 변경합니다.

즉, 해당 서비스가 사용하는 모든 민감정보를 하나의 JSON으로 key-value로 저장하여 제공합니다.

예시:

```yaml
{
  "DB_PASSWORD": "...",
  "REDIS_HOST": "...",
  "REDIS_PASSWORD": "..."
}
```

**추가 설명 - GCP Secret Manager와 K8s Secret

우리는 보안을 위해 GCP Secret Manager를 쓰기로 했지만, K8s 안에 있는 애플리케이션(Pod)은 GCP 바깥세상을 모르는 게 좋습니다. (Loose Coupling) 따라서 아래와 같은 흐름으로 민감정보들이 전달됩니다.

- GCP Secret Manager (진짜 금고): gaia-user-api-d라는 이름으로 진짜 비밀번호가 저장된 곳, 클라우드 영역.
- External Secrets Operator (배달원): GCP금고에서 주기적으로 비밀번호를 복사해옵니다.
- K8s Secret (택배): 배달원이 클러스터 내부에 만들어 둔 복사본입니다. 애플리케이션과 같은 네임스페이스에 존재합니다.
- Pod: GCP까지 갈 필요 없이, K8s Secret을 열어 환경변수로 비밀번호를 읽어 와 본인 application에 주입합니다.

참고로 개발자는 GCP Secret Manager만 관리하며, K8s Secret은 자동으로 생성되는 artifact입니다.

**logical database naming convention 관련 부연 설명

예시를 들어 기존의 마이크로서비스들을 마이그레이션 할 때에 어떻게 처리되는지에 대한 이해를 도우려 합니다.

case 1) crm-main-fastapi

- AS-IS 분석
    - crm: scope는 명확함
    - main: BAD. 너무 포괄적이라 DB 이름이 main이 되어버림. 이 서비스가 다루는 핵심은 "병원 운영/진료/마케팅 기초 데이터"임.
    - fastapi: BAD. 기술 스택은 이름에서 제거해야 함 (앞서 설명됨)
- TO-BE 적용
    - Scope: crm
    - Topic: main 대신 비즈니스 실체를 의미하는 “clinic” 사용
    - Function: REST API 제공 백엔드 서버이므로 “api” 사용
- 변환 결과
    - Github Repo: service-crm-clinic-api
    - Full Name: crm-clinic-api
    - Short Name: crmclncapi
    - Logical DB Name: clinic
    - DB User: crmclncapi

case2) consult-session-orchestrator

- AS-IS 분석
    - consult-session: 나쁘지 않지만, 너무 길다. 특히 뒤에 orchestrator까지 붙어서
    - orchestrator: 역할이긴 하지만, 결국 API 요청을 받아 처리하는 서버라면 표준 Function인 api가 맞음
    - 역할: 피부 분석, 결과 조회 등 consultation의 전 과정을 다룸
- TO-BE 적용
    - Scope: dermacode
    - Topic: session, analysis 등을 모두 아우르는 상위 개념인 consult 채택 (만일 분석만 담당한다면 analysis 등을 사용할 수도 있겠지만, 이는 모든 기능을 아우르지 못하며, db의 이름을 비직관적으로 만들도록 해버림)
    - Function: api
- 변환 결과
    - Github Repo: service-dermacode-consult-api
    - Full Name: dermacode-consult-api
    - Short Name: dcconsapi
    - Logical DB Name: consult
    - DB User: dcconsapi

**데이터 및 스토리지 리소스 (Unique Constraint)**
전 세계(Global) 혹은 프로젝트 내에서 유일한 이름을 가져야 하므로 Prefix(dra)가 필수입니다.

|  | 조합 규칙 (Format) | 예시 (Example) | 비고 |
| --- | --- | --- | --- |
| GCS Bucket | {Prefix}-{Env}-{Region}-{Short Name} | Private: dra-d-seo-gauserapi
Public(CDN): dra-p-seo-assets | **전역 유일성 필수.** 가장 충돌이 잦은 리소스이므로 Full Path 사용. |
| Artifact Registry | {Prefix}-{Env}-repo | dra-d-repo | 도커 이미지는 보통 프로젝트/환경 단위 **통합 저장소**를 사용하고, 내부에서 이미지 이름(gaia-user-api)으로 구분합니다. |
| Cloud SQL Instance | {Prefix}-{Env}-{Region}-sql-{Identifier}
{Prefix}-{Env}-{Region}-sql-{Identifier}-rep{#} | dra-d-seo-sql-main
dra-p-seo-sql-main
dra-p-seo-sql-main-rep1 | **(Dev)** 1개 (sql-main)
**(Prod)** Master(sql-main) + Replica(sql-main-rep1)
replica는 read only replica를 의미하며, 부연설명은 하단에 있음 |

**GCS Bucket 단위 관련 변경점

- 변경의 핵심: 폴더 격리 → 버킷 격리
    - AS-IS: dev, staging - cluster(environment) 별로 하나의 버킷 내부에 서비스 별 폴더명으로 격리
    - TO-BE: Application 및 Environment 별로 하나의 독립된 버킷을 생성
- 왜 버킷을 쪼개나요?
    - 비용: GCS의 과금 모델은 저장된 데이터의 용량 및 트래픽 기준, 즉 생성 비용은 없음.
    - 보안 격리 및 사고 방지(Blast Radius 최소화): IAM 설정을 세밀하게 걸어둘 계획
    - 수명 주기 관리 최적화: 서비스 별로 만료 정책(예를 들어 로그와 영구저장사진)을 다르게 설정할 수 있음
- 버킷을 사용하지 않는 서비스는?
    - 우리의 infra-terraform-main 모듈은 기본적으로 버킷을 생성하지 않도록(enabled=false) 설정하고, 필요한 서비스만 true로 켜는 방식을 권장합니다.
    - 즉, 파일을 다루는 서비스는 규칙에 따라 전용 버킷 생성, 파일을 다루지 않는 서비스는 아무것도 만들지 않음
- Legacy Proxy 서버의 제거
    - 기존에 GCS 접근 제어를 위해 사용되던 storage-gin은 새로운 아키텍처로 인해 deprecate됩니다.
- 용도 별 Buckets
    - Private Buckets
        - 대상: 대부분의 마이크로서비스
        - 구조: 1 service + env = 1 bucket
        - 접근 방식: Signed URL
    - Public Asset Bucket
        - 대상: assets (로고, 아이콘, 폰트, 상품 썸네일)
        - 특징: 누구나 봐도 되고, 웹페이지에서 빨리 로딩되는 것이 최우선
        - 구조: 전사 통합 1개 Bucket (dra-p-seo-assets, dra-d-seo-assets)
        - 접근 방식: Public URL + CDN(이 하나의 버킷에만 cloud cdn 연결)
- Terraform 운영 주의사항 (매우 매우 중요)
    - Terraform 코드에서 create_bucket = true를 false로 변경하는 행위는 단순히 ‘관리를 중단하겠다’는 의미가 아니고, 리소스 파괴를 의미함 (데이터 영구 손실)
    - 데이터를 살리고 Terraform 관리만 끊고 싶다면, 코드를 수정하기 전에 반드시 State 제거를 먼저 수행해야함.
    
    ```yaml
    terraform state rm module.gaia_user_api.google_storage_bucket.this
    ```
    
    - 우발적인 실수로 인한 치명적인 데이터 손실을 막기 위해, 시스템 레벨에서 몇 가지 안전장치를 세울 계획입니다.
        - Terraform force_destroy=false (default): Terraform 모듈의 기본 설정으로, 버킷 내부에 파일(객체)이 단 하나라도 존재하면 Terraform은 삭제 명령을 거부하고 에러를 발생시킴(Error: Bucket is not empy. Terraform cannot destroy it.” 즉, 개발자 실수로 인한 apply를 1차적으로 막아줌.
        - Terraform prevent_destroy: 코드 레벨에서 삭제 방지 태그를 붙일 수 있습니다. 이를 활용하면 terraform plan 단계에서부터 "삭제할 수 없는 리소스입니다"라고 경고하며 진행을 차단합니다.
        - GCP Soft Delete (last resort): 모든 GCS 버킷에는 Soft Delete 정책이 기본 적용되며, 1/2단계를 뚫고 버킷이 실제로 삭제되었더라도 삭제 후 7일(기본값) 동안은 복구 가능한 상태이며, gcloud storage buckets restore … 명령어로 데이터 손실 없는 복원이 가능합니다.

**Cloud SQL Instance 관련

**변경의 핵심: Scale-up 전략과 Logical Isolation**

- Topology 전략: 물리적 통합, 논리적 분리
    - B2B Saas 트래픽 특성상 비용 효율을 위해 고성능 단일 인스턴스 전략 활용
        - Dev: 비용 절감을 위해 1개 인스턴스로 모든 DB 통합(기존과 동일)
        - Prod: Master(1개) + Read Replica (N개) 구조로 운영
    - MSA 원칙 준수: 물리 서버는 공유하지만 1 Service = 1 Logical DB 원칙을 엄격하게 적용함. 각 서비스는 자신에게 할당된 DB 이름(Topic)과 계정(Short Name)만 사용하므로, 물리적으로 통합되어 있어도, 타 서비스의 Data에 접근하는 것은 불가능합니다.
- Read/Write Splitting
    - Production 환경에는 Read-Only Replica를 둘 계획입니다. 서비스 안정성을 위해 트래픽을 의도적으로 분리해야 합니다.
        - Master(-main): 데이터 변경(INSERT, UPDATE, DELETE) 및 일반적인 조회 트랜잭션
        - Replica(-main-rep1): 무거운 조회 쿼리 전용
    - 코드가 dev, production에 따라 다르게 설정하는 것은 번거롭지 않을까요?
        - 코드는 환경변수로 제어하여 변하지 않습니다.
        - 해결 방법: 환경 변수 매핑 (Configuration Injection) - 애플리케이션은 항상 두 개의 접속 정보(Writer, Reader)를 요구하도록 코딩합니다.
        
        코드 (Config)
        
        ```python
        # 항상 두 개의 URL을 받도록 설정
        WRITE_DB_URL = os.getenv("DB_WRITE_URL")
        READ_DB_URL = os.getenv("DB_READ_URL")
        ```
        
        Dev 환경 (k8s values.yaml): 둘 다 같은 곳을 가리키게 셋팅
        
        ```yaml
        env:
          - name: DB_WRITE_URL
            value: "jdbc:postgresql://dra-d-seo-sql-main..."
          - name: DB_READ_URL
            value: "jdbc:postgresql://dra-d-seo-sql-main..."  
            # <- 그냥 Master를 가리킴
        ```
        
        Prod 환경 (k8s values.yaml): 서로 다른 곳을 가리키게 셋팅
        
        ```yaml
        env:
          - name: DB_WRITE_URL
            value: "jdbc:postgresql://dra-p-seo-sql-main..."      # Master
          - name: DB_READ_URL
            value: "jdbc:postgresql://dra-p-seo-sql-main-rep1..." # Replica
        ```
        
    - 유저의 권한도 제가 sql script로 관리해야 하나요?
        - DB/User/Grant 생성은 DevOps(Terraform)의 영역이고, 테이블(Schema) 생성은 서비스 개발자(ORM)**의 영역입니다. 따라서 생성된 DB, User를 사용하여 구현에만 집중해주시면 됩니다.
        - 올바른 Role & Responsibility 예시
            - 1단계: 인프라 프로비저닝 (주체: DevOps 개발자 + Terraform)
                - Instance 생성: sql-main 물리 서버 생성
                - Logical DB 생성: CREATE DATABASE user; (텅 빈 DB)
                - DB User 생성: CREATE USER gauserapi WITH PASSWORD ‘asdf’;
                - 권한 부여(Grant): GRANT ALL PRIVILEDGES ON DATABASE user TO gauserapi;
                - 결과: user라는 빈 DB가 있고, gauserapi/asdf로 접속 가능한 상태
            - 2단계: 스키마 관리 (주체: Layer 3 개발자 + ORM)
                - 주체: service-gaia-user-api (Flyway, Alembic, Gorm AutoMigrate)
                - 앱이 gauserapi 계정으로 user DB 접속
                - Table, Index 생성 등등
- PgBouncer (Transaction Pooling)
    - 해당 부분은 다른 섹션에서 심도깊게 다룹니다.
    - 배경: MSA 환경에서 K8s 파드가 오토스케일링되면 DB 커넥션 수가 폭증해 DB가 다운될 위험이 있습니다. 특히, ORM을 사용하여 커넥션 낭비가 심합니다.
    - 정책: 애플리케이션은 Cloud SQL에 직접 붙지 않고, PgBouncer(예시 서비스명 - service-gaia-db-proxy)서비스를 통해 접속하는 것을 강제합니다. 이를 통해 커넥션을 효율적으로 재사용(Pooling)할 수 있으며 Master/Replica 라우팅 안정성도 높여줍니다.
    - 단일 Instance를 사용할 계획이므로 PgBouncer 도입은 선택이 아닌 필수입니다.
    - Connection Pooling이 아닌 Transaction Pooling으로 확정짓습니다.
    - 개발자 숙지 사항(뒤에서 더 다룰 예정)
        - Session 레벨의 기능 (Temp Table, Advisory Lock, Prepared Statement)사용에 주의해야 합니다.
        - ORM 설정에서 Prepared Statement 기능 비활성화를 강하게 권장합니다.

**인프라 및 네트워크 (Layer 1, 2)**

애플리케이션 개별이 아닌, 전체 시스템을 지탱하는 기반 리소스입니다. Scope 대신 인프라 용어가 들어갑니다.

| 리소스 (Resource) | 조합 규칙 (Format) | 예시 (Example) | 비고 |
| --- | --- | --- | --- |
| VPC Network | {Prefix}-{Env}-vpc | dra-d-vpc
dra-p-vpc | 단일 프로젝트 내에서도 환경별로 네트워크를 격리해 보안 사고 전파를 방지합니다. |
| Subnet | {Prefix}-{Env}-{Region}-sbn-{Tier} | dra-d-seo-sbn-pri
dra-p-seo-sbn-db | 하단의 Tier 정의를 참고해주세요 |
| GKE Cluster | {Prefix}-{Env}-{Region}-gke | dra-d-seo-gke
dra-p-seo-gke | 클러스터는 환경/리전 당 1개를 원칙으로 합니다 |
| Static IP | {Prefix}-{Env}-{Region}-{ip}-{Usage} | dra-p-seo-ip-ingress | ingress, NAT 등 고정 IP 식별용 |
| Firewall Rule | {Prefix}-{Env}-fw-{Rule} | dra-d-fw-allow-internal | 방화벽 규칙 이름 |

**Subnet Tier 정의 (용도 구분)

- 서브넷 이름의 맨 뒤에 붙는 {Tier}는 다음 규칙을 따릅니다.
    - pub(Public): 외부 로드밸런서 (ALB) 등이 위치하는 공용망
    - pri(Private): GKE 노드와 파드가 위치하는 사설망 (외부 직접 접속 불가능)
    - db(Database): Cloud SQL, Redis 등 데이터 스토어가 위치하는 데이터망 (Private Service Access)

**해당 다섯 가지 리소스에 대한 기초 설명

GKE Autopilot은 구글이 네트워크나 노드 관리는 우리가 할 테니, 당신은 파드만 신경쓰세요라고 추상화해둔 서비스입니다. 하지만 테라폼으로 인프라를 직접 통제하고 GKE Standard로 넘어가려면 이제 네트워크를 직접 설계해야합니다.

- VPC Network: 우리 시스템만의 독립된 사설망
- Subnet: 우리 Network를 용도별로 나눈 구역 (보안 격리)
    - pub: 로드밸런서가 여기 삽니다. 외부연결 O
    - pri: API 서버가 여기 삽니다. 외부연결 X, 안전
    - db: DB가 사는 곳
        - 질문: 엥, db는 cloud sql에 있는거아니였어? cloud sql은 구글이 관리해주는 PaaS인데?
        - 답변: 물리적으로는 구글 땅에 살지만, 주소(IP)는 우리 건물 주소를 씀.
            - 왜 db 서브넷이 필요한가요? 만약 Cloud SQL을 우리 VPC와 연결하지 않는다면 DB에 접속하기 위해 Public IP를 사용해야 함. 있을 수 없는 일이고 Private IP로만 통신해야 함.
            - PSA(Private Service Access): 내부 통신을 하려니 문제가 생김. DB는 구글 VPC에 있고, API 서버는 우리 VPC에 있음. 그래서 우리 VPC의 IP 주소 중 일부를 구글에게 빌려주는 형식임 (헤이 구글, 내 DB 만들 때 내 네트워크의 이 주소 대역-db subnet을 사용해서 만들어줘). 이렇게 하면 물리적으로는 구글 데이터센터 어딘가에 박혀있지만, 네트워크상으로는 마치 우리 옆방에 있는 것처럼 접속할 수 있게
- Firewall Rule: 누가 어디로 들어갈 수 있는지 정하는 규칙입니다. 예를 들어 pub에서 pri로 가는 문은 열어두되, db로 바로 가는 문은 잠가라와 같은 접근 제어
- Static IP: 고정된 IP 주소로, 클러스터를 껐다 켜도 주소가 변경되면 안됨, DNS를 연결하기 위함.
- GKE Cluster: 실제 일을 하는 Node(서버)와 Pod(애플리케이션) 그룹으로, 직원들을 어느 건물(VPC), 어느 방(Subnet)에 배치할지 정해줘야 함.

### 1.3. AI 시스템 관련 (AI System Standards)

AI/ML 시스템은 일반적인 웹 애플리케이션과 라이프사이클이 다르기 때문에 코드와 모델(Artifact)를 엄격하게 분리하여 관리하는 MLOps 표준을 정의합니다.

**아키텍처 원칙: Cartridge Pattern**

우리는 API 서버를 본체(Console), AI 모델을 카트리지(Cartridge)로 취급합니다.

- Decoupling: Docker 이미지 안에 수백 MB짜리 모델 파일을 포함하지 않습니다.
- Dynamic Loading: 서빙 API가 시작될 때, GCS Model Registry에서 지정된 버전의 모델을 다운로드하여 메모리 위에 올립니다.
- 이점: 모델을 업데이트하기 위해 API 서버 코드를 수정하거나 다시 빌드할 필요가 없습니다.

**리소스 구성 및 명명 규칙 (AI Naming Matrix)**

AI 시스템은 학습(Training) → 저장(Registry) → 서빙(Serving)의 3단계로 구성되며 ,각 단계별 리소스 명명은 다음과 같이 합니다.

| 단계 | 리소스 종류 | Naming Rule | 예시 | Physical Location |
| --- | --- | --- | --- | --- |
| 1. 학습 (Learner) | Training Repo | service-{Scope}-{Topic}-train | service-dermacode-skin-train | Github
(배포되지 않는 실험용 코드) |
| 2. 저장 (Artifact) | Model Artifact | model-{scope}-{Topic} | model-dermacode-skin-analysis | GCS Bucket
(…-models) |
| 3. 서빙 (Player) | Serving API | service-{Scope}-{Topic}-api | service-dermacode-consult-api | K8s Pod
(모델을 로드해서 쓰는 주체) |
- Topic의 분리:
    - Serving API는 비즈니스 프로세스(consult)를 따릅니다
    - Model/Training은 기술적 기능(skin-analysis)를 따릅니다
    - 이유: skin-analysis model은 나중에 app 등 다른 곳에서도 재사용될 수 있기 때문에 특정 업무(consult)에 종속된 이름을 피합니다.

**Model Registry 구조 (GCS)**
모델 파일(.pt, .onnx 등)은 전사 통합 모델 버킷에서 관리합니다.

- Bucket Name: {Prefix}-{Env}-{Region}-models (예: dra-p-seo-models)
- Directory Structure

```yaml
dra-p-seo-models/
├── dermacode/                  # [Scope]
│   ├── skin-analysis/          # [Topic] - 모델 식별자
│   │   ├── v1.0.0/             # [Version] - Semantic Versioning
│   │   │   ├── model.pt
│   │   │   └── config.json
│   │   └── v1.0.1/
│   └── pore-detection/
└── dentalcode/                 # [Future Scope]
```

** 무거운 모델(예를 들어 Transformer)은 어떻게 하나요?

TFLite같은 가벼운 모델들은 서버가 뜰 때 다운로드 받아도 수 초 안에 끝나서 문제가없습니다. 하지만 BERT나 Stable Diffucison 같은 거대 모델은 다운로드에만 몇 분이 소요될 수 있습니다. 하지만 이 때에도 도커 이미지에 모델을 굽지 않는다는 대원칙은 변함없습니다. 대신 Cartridge를 꽂는 방식을 바꿉니다. (이는 추후에 고려해도 될 대상임) 예를 들어 다음과 같은 패턴들을 고려할 수 있습니다. (후자로 진행할 예정)

- Init Container: 본체(API 서버)가 뜨기 전에 init-container가 먼저 GCS에서 모델을 다운로드하여 공유 볼륨(EmptyDir)에 저장해둡니다. 이렇게 하면 API 서버 코드는 다운로드 로직을 몰라도 그냥 특정 폴더에 모델이 있다고 가정하고 뜹니다.
- GCS Fuse / PV Mount: 다운로드 받는 것이 아니고, GCS Bucket 자체를 로컬 폴더처럼 Mount 해버립니다. 이는 GKE의 Cloud Storage FUSE CSI driver를 사용하여 구현할 수 있습니다. 이를 활용하면 파드가 뜨는 시간이 0초(네트워크 드라이버로 잡히니까), 읽는 시간은 최초 추론 시에만 스트리밍으로 읽어와야 해 약간 느리고 이후에는 캐싱되어 매우 빠름
    - https://docs.cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/cloud-storage-fuse-csi-driver?hl=ko
    - https://docs.cloud.google.com/kubernetes-engine/docs/how-to/cloud-storage-fuse-csi-driver-setup?hl=ko#standard

### 1.4. 브랜치 전략 및 환경 매핑 (Branching Strategy)

목표: 코드는 main으로 흐르고, 배포는 GitOps로 통제합니다. 복잡한 환경별 브랜치(dev, staging 등), feature 별로 추가된 난잡한 좀비 branch들을 제거하고, 단일 Single Source of Truth를 유지합니다. 이를 위해 Best-Practice로 꼽히는 Trunk-Based Development 및 GitOps로 전략을 전환합니다.

**Strategy Shift: AS-IS vs To-BE**

왜 익숙한 방식을 버리고 새로운 방식을 도입해야 하는지에 대한 배경입니다.

| 항목 | Legacy | New-Standard | 기대 효과 |
| --- | --- | --- | --- |
| 기준 브랜치 | dev, staging (파편화됨), main은 아무 의미없음 | main 브랜치가 Single Source of Truth | 모든 개발자는 main만 바라봅니다. 코드 통합의 복잡도가 사라집니다. |
| 브랜치 관리 | Issue 종료 후에도 삭제 안되고, 수백 개의 좀비 브랜치가 누적됨 | 즉시 삭제 (Delete on Merge) | PR이 Merge되는 순간, Github이 자동으로 브랜치를 삭제해 저장소를 깨끗하게 유지합니다. |
| 배포 방식 | CI(Github Actions)가 직접 helm upgrade 수행 (Push) | ArgoCD가 변경사항 감지 (Pull) | CI 도구에게 Cluster Admin Key를 주지 않습니다. (보안 강화) |
| 빌드 전략 | 환경별로 매번 다시 빌드 (Dev 빌드 ≠ Staging 빌드) | Build Once, Deploy Selectively | 한번 빌드된 이미지를 끝까지 사용합니다. (즉, dev에서 됐는데 prod에서 안돼요 발생 X) |

**Layer 3 Application Repository 전략 (Trunk-Based)**

service-gaia-user-api와 같은 애플리케이션 저장소의 운영 규칙입니다.

- 메인 전략: Trunk-Based Development
    - dev, release, staging과 같은 Long-lived branches의 생성 및 운용을 금지합니다.
    - 오직 main 브랜치만이 유일한 최신 코드를 가지게 됩니다.
    - Environment Isolation(환경 격리): 미완성 기능도 main에 병합하여 dev 환경에 배포하는 것을 두려워하지 않습니다. Prod 환경으로 Promotion만 시키지 않으면 안전하기 때문입니다.
    - 모든 사고 (Hotfix/Rollback)조차도 (간소화된) 절차를 따릅니다.
- 작업 흐름 (Workflow)
    - Feature Branch: 개발자는 main에서 feat/something-something 브랜치를 생성하여 작업합니다.
    - Pull Request (PR): 작업이 끝나면 main으로 PR을 생성합니다.
    - CI Check: PR 단계에서 유닛 테스트와 빌드가 통과해야 합니다.
        - 유닛 테스트는 초기 구현 단계에서 미비되어있을 수 있습니다.
    - Auto Deletion: Squash&Merge 방식으로 Merge하여 병합 완료 시 브랜치는 즉시 자동 삭제됩니다.
        - 설정: Repository Settings > General > Automatically delete head branches 활성화가 모든 Repository에 대해 필수입니다.

Branch Protection Rules

main branch의 안정성을 위해 Github Repository 설정에서 다음 규칙들을 강제합니다.

- Require Pull Request: main 브랜치에 직접 Push 금지
- Require Approvals: 최소 1명 이상의 동료 리뷰 승인 필수
    - 단, 초기에는 개발 속도와의 tradeoff를 위해 자가 리뷰를 허용할 수도 있습니다.
- Require Status Checks
    - Build Success
    - Lint/Type Check
    - Unit Test - 초기 단계에는 커버리지가 낮을 수 있음, but 빌드와 린트가 깨진 코드는 절대로 머지될 수 없습니다.

**Environment Mapping & Promotion**

어떤 코드가 어디에 배포되는가에 대한 규칙입니다. 환경별 브랜치를 쓰지 않고, GitOps 저장소의 폴더와 이미지 태그로 환경을 구분합니다.

| 환경 (Env) | 매핑 대상 (Source) | 배포 트리거 (Trigger) | 이미지 태그 전략 | 비고 |
| --- | --- | --- | --- | --- |
| Dev | main Branch (App Repo) | 자동(Automatic)
main에 머지되면 즉시 배포 | Short SHA
(예시: main-a1b2c3d) | 최신 코드를 빠르게 검증하는 환경 |
| Prod | Docker Image (Artifact) | 수동 승인(Promotion)
Dev에서 검증된 이미지를 수동으로 승격 | SemVer Tag
(예시: v1.0.0) | 안정성이 보장된 이미지만 배포 |

핵심 원칙: Build Once - 배포를 위해 코드를 다시 빌드하지 않음. Dev 환경에서 충분히 테스트된 해당 이미지에 태그만 새로 달거나, 그대로 Prod 환경설정 파일에 Inject 합니다.

**GitOps Deployment Flow (ArgoCD)**

infra-gitops-manifests 저장소를 통한 배포 흐름입니다.

- Step 1: Dev 자동 배포 (Continuous Deployment)
    - 개발자가 app repo의 main 브랜치에 코드를 merge 합니다.
    - CI(Github Actions)가 도커 이미지를 빌드하고 Artifact Registry에 푸쉬합니다.
    - CI 봇이 infra-gitops-manifests 저장소의 environments/dev/services/…/yaml 파일에서 image.tag를 업데이트하고 커밋합니다.
    - ArgoCD가 Git변경을 감지하고 Dev Cluster에 즉시 반영합니다.
- Step 2: Prod 승격 배포 (Promotion)
    - 개발자/실사용자 등이 Dev 환경에서 기능을 검증합니다.
    - 배포가 결정되면, infra-gitops-manifests 저장소에서 PR을 생성합니다.
        - 내용: environments/dev의 검증된 이미지 태그를 environments/prod로 복사합니다
    - 책임자가 PR을 승인 및 Merge하면 ArgoCD가 Prod Cluster에 해당 내용을 반영합니다.
    

**왜 이렇게까지 해요?**

- 배포의 Visibility
    - 누가 언제 배포했는지를 Git Commit Log만으로 확인 가능함
    - ArgoCD UI를 통해 현재 배포된 버전과 상태를 한 눈에 파악할 수 있음
- Security
    - CI 도구에게 Cluster Access Admin을 부여안해도 되어 보안적으로 더 안전함
- Self-Healing
    - 누군가 실수로 kubectl로 서버 설정을 변경해도 ArgoCD가 Git에 정의된 바와 다르다는 것을 감지하고 원상 복구합니다 (Configuration Drift 방지)

**Branch & Commit Naming Convention**

브랜치 이름은 곧 작업의 성격과 일치해야만 합니다. 우리는 {Type}/{Issue-Number}-{Description}의 형식을 따르며, Whitelist 방식으로 허용된 Type만을 사용할 수 있습니다.

- 형식: {type}/{issue-number}-{description}
- 예시: feat/238-login-page, fix/438-payment-error

**Type Whitelist(허용된 말머리)

| Type | 설명 (Description) | 사용 예시 | 비고 |
| --- | --- | --- | --- |
| feat | 새로운 기능(feature) | feat/123-login-page | 가장 많이 쓰임, 제품에 새로운 피처를 더함 |
| fix | 버그 수정 (bug fix) | fix/239-consult-error | 오동작 수정 |
| refactor | 리팩토링 | refactor/12-user-service | 기능 변경 없이 코드 구조만 개선함 |
| docs | 문서 수정 | docs/485-readme-update | README나 md 문서, 주석 수정 |
| style | 포맷팅, 스타일 | style/33-lint-fix | 코드 로직 변경 없이 들여쓰기, 세미콜론 등 수정 |
| test | 테스트 코드 | test/88-add-payment-unit-tests | 테스트 수정 및 추가 (production code 변경 없음) |
| ci | CI/CD 설정 | ci/83-github-actions | cicd |
| chore | 기타 잡무 | chore/921-update-deps | 빌드 스크립트 수정, 패키지 버전 올리기  |
| hotfix | 핫픽스 | hotfix/33-add-random-button |  |
| rollback | 롤백 | rollback/38-rollback-to-branch-37 |  |

**Hotfix Process (긴급 장애 대응)**

Prod 환경에 치명적인 버그가 발생했을 때에도 파이프라인을 우회하지 않습니다. 대신, 빠르게 수정할 수 있는 절차를 마련합니다.

- Branch: main에서 hotfix/{issue_id}-{description}을 생성합니다
- PR & Review: Review는 생략하되 PR은 생성합니다.
- Dev Auto-Deploy: main 머지 후 Dev 환경 배포 확인
- Fast Track Promotion: 확인 즉시 infra-gitops-manifests에서 Prod 승격 PR 생성 및 머지 (Review는 생략될 수 있습니다. 추후 Merge된 PR에서 code difference를 재차 점검합니다.)

**Rollback Strategy (장애 복구)**

배포 직후 장애 발생 시의 복구 절차입니다.

- 원칙: ArgoCD UI의 복구 버튼을 사용하지 않고, Git Revert를 사용합니다.
- 절차
    - infra-gitops-manifests 저장소에서 방금 머지된 승격 커밋을 Revert하는 PR을 생성합니다.
    - 즉시 Merge하여 ArgoCD가 이전 버전 이미지로 되돌려지도록 함.

**Governance Automation (Templates)**

규칙 준수를 돕기 위한 Github Template들을 설정합니다.

- Issue Template: 이슈 생성 시 자동으로 로드되는 markdown 양식입니다.
- Pull Request template: PR 생성 시 개발자가 체크해야할 리스트입니다.

### 1.5. 접근 제어 및 권한 관리 (RBAC & IAM)

목표: 모든 사용자와 애플리케이션(Workload)은 업무 수행에 필수적인 최소한의 권한만을 부여받습니다. 특히, Workload Identity를 도입하여 보안 키(JSON Key) 관리에 따른 리스크를 원천 차단합니다.

**접근 제어 철학**

- Human
    - 원칙: No Production Write Access
    - 인프라 변경은 오로지 Terraform, ArgoCD를 통해서 수행합니다. 개발자는 콘솔에서 보기(View) 권한만을 가집니다.
- Machine
    - 원칙: Keyless Authentication
    - 장기 생존하는 JSON Key 파일 사용을 금지합니다. 대신, 구글이 보증하는 Workload Identity를 통해 짧은 수명의 토큰을 발급받아 통신합니다.

**Human Access: GCP IAM & Kubernetes RBAC**

사람이 클라우드 리소스와 클러스터에 접근하는 규칙입니다. Google Groups를 활용하여 역할을 구분 및 관리합니다.

- GCP IAM Roles (Cloud Resource): GCP 콘솔 접근 권한입니다.

| 그룹명 (Group) | 역할 (Role) | 권한 범위 대상 | 권한 |
| --- | --- | --- | --- |
| gcp-org-admins | Owner | 리드 개발자, DevOps 개발자 (최소인원 지향) | IAM, Network, Billing 등 모든 리소스 |
| gcp-developers | Viewer | 모든 개발자 | 리소스 조회, Cloud Logging, Monitoring Dashboard Access |
| gcp-billing | Billing Admin | 현재: X
추후 생긴다면: 재무/회계 담당자 | 결제 정보 조회 및 인보이스 다운로드 |
- Kubernetes RBAC (Cluster Access): 개발자가 kubectl을 통해 클러스터에 접근할 때 적용되는 권한입니다

| 역할 (Cluster Role) | 대상 (Subject) | 허용된 작업 (Permissions) | 제한 사항 (Restriction) |
| --- | --- | --- | --- |
| cluster-admin | DevOps Team, ArgoCD Controller | - 모든 리소스(Secret 포함)의 생성/삭제/수정
(주의) CI Bot (Github Actions) 에게는 이 권한을 부여하지 않습니다. CI는 오직 Git Repo와 Artifact Registry에만 접근합니다 |  |
| developer-view | Developers | - get, list, watch: 파드 상태 조회
- logs: 로그 확인
- port-forward: 로컬 디버깅용 포트 포워딩 | - exec: 파드 내부 쉘 접속 금지
- delete, edit: 리소스의 수동 변경 금지 (ArgoCD와의 상태 불일치 방지)
- secrets: 민감 정보 조회 금지 |

- Machine Access: worload Identity Federation (keyless)
    - 애플리케이션(pod)이 인프라 리소스에 접근하는 표준 방식입니다.
    - Track 1: Google Native Resources (Direct Access) - GCS, Cloud Vision API 등 구글 서비스를 이용할 때: 애플리케이션 코드가 Google SDK를 통해 직접 인증합니다.
    - Track 2: Non-Google Secrets (Indirect via ESO) - REdis Password, API Keys 등 단순 문자열 비밀번호를 이용할 때: ESO(External Secrets Operator)가 Workload Identity를 신분증으로 GCP Secret Manager에서 비밀번호를 꺼내오고, 이를 K8s secret으로 변환하여 파드에 환경변수로 주입합니다. 애플리케이션은 GCP의 존재를 몰라도 되며(Loose Coupling), 단순히 환경변수를 읽는 것으로 개발이 끝납니다.
- Database Access Strategy
    - Application to Database
        - 경로: Pod → PgBouncer → Private Service Access (PSA) → Cloud SQL
        - 인증:
            - DB User {Short Name}
            - DB Password: ESO가 주입해준 K8s Secret 사용함
    - Human to Database (운영자 접속)
        - 원래는 IAP Tunneling을 해야하나 (Bastion Host), 기존 데이터 이관을 위해 port forwarding 된 pg admin을 통할 수도 있음 (미정)

---

# 2. 물리 인프라: 네트워크 및 보안 (Physical Infra: Network & Security)

목표: Terraform 으로 구현할 Network Topology 상세 정의

### 2.1. VPC 및 서브넷 설계 (VPC Design)

VPC란 Google Cloud라는 거대 공용 공간 속에 우리 회사만의 독립된 네트워크 경계선입니다. GKE Autopilot을 사용할 때에는 구글이 미리 만들어둔 공용 네트워크(Default VPC)의 일부분을 임대해 사용했지만, Standard환경에서는 보안과 확장성을 위해 우리가 직접 통제 가능한 독립된 사옥을 짓습니다.

**2.1.1. VPC (Virtual Private Cloud) 구성**

설정 명세(Configuration Speculations)

- Name: {Prefix}-{env}-vpc
- Routing Mode: Global
- Subnet Creation Mode: Custom (auto_create_subnetworks = false)

상세 설계 논리 (Design Rationale)

- Routing Mode는 왜 Global인가요?
    - 설정: VPC 내부 라우팅 범위를 Regional 단위가 아닌 Global 단위로 설정합니다.
    - 이유: 현재는 서울(asia-northeast3)리전만 사용하지만, 추후 DR(재해 복구)나 글로벌 확장을 위해 일본 중국 등 리전을 추가할 수 있습니다.
    - 효과: Global로 설정하면 리전이 달라도 구글의 전용 광케이블망을 통해 내부사설 IP로 안전하고 빠르게 통신할 수 있습니다 (공용 인터넷망을 타지 않음)
- Auto Create Subnetworks: 왜 false(끄기) 인가요?
    - 기본값의 위험성: true로 설정하면 VPC 생성 즉시 구글이 전세계 모든 리전에 자동으로 서브넷을 생성해버립니다.
    - 끄는 이유
        - 서울에만 서비스하는데 런던 리전 등에 네트워크가 열려있을 필요가 없음
        - 관리의 명확성: 초기에 텅 빈 VPC를 생성해두고 terraform 코드를 통해 우리가 사용하는 서울 리전에만 명시적으로 서브넷을 생성해 관리 범위를 확실하게 통제

**2.1.2. Subnet Tier 및 CIDR 설계**

개요: 용도별  보안 구획(Zoning)과 토지 크기(Sizing) 설정. VPC라는 거대한 사유지를 용도에 따라 세 가지 보안 등급인 Public, Private, Database로 나눕니다. 이를 Subnet이라고 합니다. 또한 각 서브넷이 가질 수 있는 IP 주소의 개수 (CIDR)를 넉넉하게 설계해 향후 서비스의 급격 성장 시에도 인프라 공사를 다시 하지 않도록 합니다.

명명 규칙(Naming Convention)

- {Prefix}-{Env}-{Region}-sbn-{Tier} (예시: dra-d-seo-sbn-pri)

상세 설계 (Detailed Design)

| Tier (등급) | 서브넷 이름 (예시) | CIDR (크기) | IP 개수 | Secondary Ranges (GKE only) | 용도 및 특징 |
| --- | --- | --- | --- | --- | --- |
| Public | …-sbn-pub | 10.0.0.0/24 | 256개 | N/A | - 로비의 역할을 함
- 외부와 연결되는 로드밸런서(Ingress)가 위치함
- 유일하게 외부 인터넷 트래픽을 받는 구간임 |
| Private | …-sbn-pri | 10.10.0.0/16 | 65536개 | Pod: 10.11.0.0/16
Svc:10.12.0.0/20 | - 사무실의 역할을 함
- GKE Node와 Pod(애플리케이션)들이 위치함
- 외부 IP가 없으며, 로비를 통해서만 접근 가능함 |
| Database | …-sbn-db | 10.20.0.0/16 | 65536개 | N/A | - 금고의 역할을 함
- Cloud SQL, Redis(cloud memorystore) 등 관리형 서비스 전용
- Private Service Access (PSA)를 위한 예약 공간
- 모든 Stateful 리소스를 이곳에 격리할 계획 |
| Mgmt | …-sbn-mgmt | 10.255.0.0/28 | 16개 | N/A | - 경비실
- Bastion Host (점프 서버) 전용 초소형 공 |

심층 설명

- CIDR이 무엇인지?
    - CIDR = Classless Inter-Domain Routing: IP 주소의 범위를 나타내는 표기법
    - /뒤의 숫자가 작을 수록 IP 개수는 기하급수적으로 많아짐
    - 클라우드 내부망(Private IP) 사용료는 무료임. 애매하게 잡기보다 처음부터 /16으로 최대 크기를 확보하기로 결정
- 왜 Private Subnet은 IP를 6만개나 두는지?
    - K8s의 IP Consumption
        - Kubernetes는 노드(서버)뿐만 아니라 Pod 하나하나마다 고유한 IP를 부여함
        - 노드 1대에 파드 20개 = IP 21개
    - 확장성 대비
        - 지금은 파드가 50~100개지만, 마이크로서비스가 100개로 늘어나고, 트래픽 폭주로 오토스케일링 발생하면 파드 충분히 수 천 개 될 수 있음
        - /24 같은 소규모 대역 쓰면 IP 고갈해 Cannot Schedule Pod 뜰 수 있음
    - 결론: pri 서브넷은 애플리케이션이 살기 때문에 일단 크게 잡고 보는 것
- private, public, databse의 3-tier 격리를 하는 이유
    - 보안: 해커가 운좋게 public 서브넷 로드밸런서 뚫고 들어오면 Private 서브넷에 있는 API 서버에는 도달할 수 있어도 그 뒤의 db 서브넷에는 직접 닿을 수 없음 (방화벽 규칙에 의해)
    - 만약 모든 서버가 한 서브넷에 위치하면 로비가 뚫리는 순간 금고까지 프리패스
- Redis, 이거 원래 우리가 bitnami든 뭐든 pod로 직접 띄우지 않았었나요?
    - 안정성, 효율성 면에서 GCP 제공 Google Cloud Memorystore for Redis API를 사용하고자 함.
        - 이유 1: 생명 주기 분리 (Decoupling Lifecycle)
            - AS-IS (Redis as Pod): K8s cluster와 redis가 운명 공동체임. Cluster 점검 시 redis pod도 재시작 = 로그인 된 모든 유저 세션이 풀리고 캐시가 날아감
            - TO-BE (Memorystore): Redis는 k8s가 죽든말든 노드를 갈아엎든말든 외부에 별도로 살아있음 (cloud sql처럼). 이 것이 Stateless Architecture에 보다 부합함.
        - 이유 2: OOM 리스크 격리
            - AS-IS: 아직은 사용량이 적어 상관이 없지만, redis는 메모리 먹는 괴물인데 redis가 메모리 과도하게 써서 노드 다운시키면 (Node OOM), 같은 노드에 있던 멀쩡한 API 서버들도 같이 물귀신해버림
            - TO-BE: Redis가 메모리가 터져도 구글이 관리하는 VM에서 터지는거라 우리 API 서버에 피해를 주지 않음
        - 이유 3: 인력 시소스 절감
            - AS-IS: 왜 죽었는지 디버깅하고 고가용성 구성하려면 귀찮음
            - TO-BE: 구글이 가동률 보장
    - 이거 호스팅해버리면 대시보드 UI는 이제 못쓰나요?
        - Bastion Host로 Memorystore를 tunneling한 뒤 TablePlus든지 Redis Desktop Manager든지 아무거나랑 연결해 사용 가능
- mgmt 서브넷은 뭔지?
    - 한줄 비유: 반도체 공장 등에 출입 하기 전에 통과해야하는 에어 샤워실 같은 것.
    - 우리는 보안을 위해 API 서버(pri)와 데이터베이스(db)를 사설 서브넷(private subnet)에 가두어 두었음. 이 방에는 창문(public ip)가 하나도 없음!
    - 근데 우리 관리자는 어떻게 수리하러 들어가야하나요?
    - 그래서 mgmt 서브넷이라는 작은 방을 만들고, 거기에 가장 튼튼한 문이 달린 작은 컴퓨터(vm) 하나를 둠. 이게 bastion
        - 관리자는 오직 bastion으로만 들어갈 수 있음
        - 이 bastion 안에서 다시 내부의 db나 api 서버로 접속함 (점프! - 그래서 이걸 점프 서버라고 부르기도 함)
    - (참고) GCP IAP(Identity-Aware Proxy) 를 사용하면 vpn도 안깔고, 포트도 안열어두고도 bastion 사용 가능
- Node IP 말고 왜 IP 대역이 2개나 더 필요한지?
    - GKE Standard는 VPC-Native 모드로 동작함. 이는 파드가 가짜 네트워크(Overlay)가 아닌 VPC 상의 실제 IP를 가진다는 뜻임. 따라서, 서브넷 안에 보조 IP 주머니(Secondary IP Range) 2개를 미리 달아두어야 함.
        - Pod Range(10.11.0.0/16):
            - 용도: 실제 애플리케이션(Pod)들이 하나씩 가져가는 IP
            - 크기: /16(65536개). 노드(Primary) 대역과 동일한 크기로 잡아, 파드가 아무리 많이 떠도 IP가 부족하지 않음
        - Service Range(10.12.0.0/20):
            - 용도: K8s Service(ClusterIP)가 사용하는 가상 IP
            - 크기: /20(4096개). 서비스(Service) 리소스는 파드만큼 많이 생성되지 않으므로 이 정도로 할당
- Terraform 적용 시 주의사항
    - 서브넷 리소스를 정의할 때 secondary_ip_range 블록을 반드시 포함해야 하며, 클러스터 생성 시 ip_allocation_policy에서 이 범위를 이름(range_name)을 참조해야 함
    - 개념 예시
    
    ```yaml
    # (개념 예시)
    resource "google_compute_subnetwork" "private" {
      name          = "dra-d-seo-sbn-pri"
      ip_cidr_range = "10.10.0.0/16"  # Node용
    
      secondary_ip_range {
        range_name    = "gke-pods"
        ip_cidr_range = "10.11.0.0/16" # Pod용
      }
      secondary_ip_range {
        range_name    = "gke-services"
        ip_cidr_range = "10.12.0.0/20" # Service용
      }
    }
    
    # (GKE Cluster 리소스 예시 - 추가)
    resource "google_container_cluster" "primary" {
      name = "dra-d-seo-gke"
      # ...
      ip_allocation_policy {
        # 서브넷에서 정의한 range_name과 정확히 일치해야 함
        cluster_secondary_range_name  = "gke-pods"
        services_secondary_range_name = "gke-services"
      }
    }
    
    ```
    

### 2.2. 네트워크 보안 및 통신 제어 (Firewall & NAT)

목표: Deny-All (기본 차단) 정책을 기반으로 꼭 필요한 통신 경로만 White-list(허용) 방식으로 뚫어줌. 또한, 외부 IP가 없는 프라이빗 서버들이 인터넷을 사용할 수 있도록 안전한 통로 (NAT)를 개설함.

**2.2.1. Cloud NAT (Outbound Traffic Gateway)**

개요: 우리의 API 서버 (pri 서브넷)는 보안을 위해 외부 IP(Public IP)가 아예 없음. 그런데 그렇다면 우리 서버가 Docker Hub에서 이미지를 다운받거나 Notification API 등을 호출 할 때에 인터넷을 어떻게 사용하는지? → 이 문제를 해결하기 위해 Cloud NAT(Network Address Translation)를 설치함.

상세 설계 및 동작 원리

- 위치: Router라는 논리적 장비에 Cloud NAT 기능을 켬
- 동작
    - 내부 서버 10.10.x.x.가 네이버에 접속하고싶어! 하고 요청을 보냄
    - Cloud NAT가 이 요청을 가로채서, 자신의 Public IP로 포장지(Source IP)를 갈아끼움
    - 인터넷 세상에 갔다 온 응답을 다시 내부 서버에게 전달해 줌
- 보안
    - Outbound: 가능 (NAT가 문을 열어줌)
    - Inbound: 불가능
- Terraform 설정 포인트
    - Router Name: {Prefix}={Env}-{Region}-router
    - NAT Name: {Prefix}-{Env}-{Region}-nat
    - Allocation Option: nat_ip_allocate_option = “MANUAL ONLY”
    - NAT Ips: google_compute_address 리소스를 먼저 생성하고, 그 ID 리스트를 연결함.
        - Note: IP 1개당 약 64,512개의 동시 연결(Port) 처리 가능. 초기 트래픽 고려하여 2개로 시작하고, 추후 모니터링을 통해 증설 결정
    - Log config: enable=true (누가 어디로 접속했는지의 감사 로그), filter = “ERRORS_ONLY” (모든 로그 쌓으면 비용 상승이므로 일단은 에러만 수집함)
- IP 할당 방식 (Dynamic vs Manual)
    - 결정: Manual (Static IP) 방식을 채택합니다.
    - 그 이유:
        - Dynamic(Auto): 설정은 편하지만, NAT 게이트웨이가 restart되거나 트래픽이 폭증해 스케일링 될 때 출구 IP가 바뀌어버림
        - Manual(Static): 미리 구매한 고정 IP(google_compute_address)를 NAT에 할당함
        - Why? PG사와 같은 외부 제휴사와 제휴할 시에 보안 이유로 접속하는 서버 IP를 화이트리스팅 해야하는데 IP가 바뀌면 안되기 때문

**2.2.2. Firewall Rules (방화벽 규칙)**

설계: 태그(Tag) 기반의 정밀 타격 & 우선순위 관리 - 단순히 IP 대역만으로 허용하면 위험하므로 Terraform에서 관리하는 Network Tags (bastion, gke-node)를 리소스에 부착하고 방화벽 규칙이 이 태그를 가진 서버에만 적용되도록 범위를 줄임

상세 규칙 명세 (Spec)

- 내부 통신 허용 (Allow Internal)
    - 10.0.0.0/8은 너무 넓어 다른 VPC나 VPN 연결 시 보안 위험
    - 설정:
        - Priority: 1000 (표준 허용 우선순위임)
        - Source Ranges: VPC CIDR 전체 (예: 10.0.0.0/16 - 우리가 정의한 땅 크기만큼만)
        - Target Tags: gke-node, bastion
        - Protocol/Port: tcp, upd, icmp (all internal)
    - 무슨 뜻? 우리 사옥 (VPC CIDR) 안에 있는 직원(Tag)들 끼리는 자유롭게 떠들던지 말던지
- IAP SSH 허용 (Allow ssh via iap)
    - 모든 VM에 열 필요 없음. GKE 노드에 직접 SSH 할 일은 거의 없음
    - 설정:
        - Priority: 1000
        - Source Ranges: 35.235.240.0/20 (Google IAP 전용)
        - Target Tags: bastion (얘만 열어줌)
        - Protocol/Port: tcp:22
    - 의미: 구글 신분증인 IAZP가 있는 사람만 경비실 문(Bastion) 열어줄 거임. GKE 노드로는 직행 불가능함
- 헬스 체크 허용 (allow health checks)
    - 설정:
        - priority: 1000
        - source ranges: 130.211.0.0/22, 35.191.0.0/16 (GCLB 전용 IP)
        - target tags: gke-node (로드밸런서는 노드만 찌른다)
        - Protocol/Port: tcp
    - 의미: health check는 안막을게~

요약:

| 규칙 이름 | Type | Priority | Source | Target Tags | Port | 설명 |
| --- | --- | --- | --- | --- | --- | --- |
| allow-internal | Ingress | 1000 | VPC CIDR (10.10.0.0/16) | gke-node, bastion | All | 내부 통신 프리패스 |
| allow-ssh-iap | Ingress | 1000 | 35.235.240.0/20 | bastion | TCP:22 | Bastion 접속용
(노드 접근 불가) |
| allow-healthcheck | Ingress | 1000 | 130.211.0.0/22
35.191.0.0/16 | gke-node | TCP:Any | 로드밸런서 헬스체크 |

### 2.3. 외부 접근 제어 (Bastion & IAP)

목표: Zero Trust Network의 구현 - 모든 데이터베이스와 클러스터는 Private IP만을 가지며, 인터넷과 단절됨. 관리자조차도 VPN 없이, 구글의 인증(IAM)을 거친 IAP 터널링을 통해서만 내부에 접근할 수 있는 단 하나의 창구(Bastion)을 만든다.

**2.3.1. Bastion Host 구성 (Jump Server)**

Bastion Host는 mgmt 서브넷에 위치하는 초소형 리눅스 서버로, 우리 Private Network로 진입하기 위한 유일한 물리적 관문.

상세 스펙

- Name: {prefix}-{env}-{region}-bastion (예: dra-p-seo-bastion)
- machine type: e2-micro같은 가장 저렴한 놈 (트래픽 처리용이 아니고 터널링용이므로..)
- OS: Minimal Ubuntu 같이 불필요 패키지 없는 것
- Public IP: None
    - Bastion 조차도 외부 IP 안가짐
- Security Options:
    - shielded_instance_config {enable_secure_boot = true}
    - metadata = {enable-oslogin = “TRUE”}
    - Disk Encryption: Google-managed Key (Default)

자동화 설정 (Startup Script) Bastion Host가 생성될 때, Terraform의 metadata_startup_script를 통해 Tinyproxy(HTTP Proxy)를 자동으로 설치하고 설정함.

- Why: 로컬 컴퓨터의 kubectl이나 웹 브라우저가 Bastion을 통해 Private GKE에 접속하기 위함
- Script 내용:
    
    ```bash
    apt-get update && apt-get install -y tinyproxy
    # 설정 변경: 포트 8888, 로컬호스트만 허용 (보안)
    sed -i 's/Port 8888/Port 8888/g' /etc/tinyproxy/tinyproxy.conf
    echo "Allow 127.0.0.1" >> /etc/tinyproxy/tinyproxy.conf
    systemctl restart tinyproxy
    ```
    

**2.3.2. IAP 터널링 전략(Identity-Aware Proxy)**

VPN 필요 없음. 구글 로그인이 VPN임. IAP가 HTTPS 기반 인증을 TCP 터널링으로 변환해줌

- 접속 메커니즘
    - 개발자가 로컬 터미널에서 gcloud compute ssh bastion 입력
    - 인증: 요청이 서버로 가는게 아니고 Google IAP 서버로 감. 여기에서 gcp-developers 그룹인가? 를 체크
    - Tunneling: 인증 통과 시 구글 내부망으로 우리 Bastion Host까지 암호화된 터널을 뚫음
    - 연결: 로컬 서버 접속하듯 Bastion 서버에 접속

**2.3.3. 실전 접속 가이드 (Operation Guide)**

실제로 개발자가 DB 접속해 쿼리 날리거나 Redis 데이터 확인하는 시나리오

- Cloud SQL 접속하기(Port Forwarding)
    - 내 [localhost](http://localhost) 포트를 bastion을 거쳐 db 포트와 연결함
    - 터널 생성 명령어
    
    ```yaml
    #예시
    # 내 컴퓨터의 54320 포트를 -> Bastion을 거쳐 -> 저 멀리 DB(10.20.0.5)의 5432 포트로 연결해줘!
    gcloud compute ssh dra-p-seo-bastion \
    	--tunnel-through-iap \
    	-- -L 54320:10.20.0.5:5432 -N -q -f
    ```
    
    - 접속
        - 이후에 내 컴퓨터 db tool (dbeaver, pgadmin 등)을 켜고, host: localhost, port: 54320 이렇게 접속하면 db 접속 가능
- Private GKE Cluster
    - GKE Control Plane (마스터 노드)도 private endpoint만 열어두면 로컬에서 kubectl 안먹힘.
    - 절차
        - bastion 접속: gcloud compute ssh dra-p-seo-bastion
        - kubectl 실행: bastion 내부에는 kubectl 및 gke-gcloud-auth-plugin이 설치되어 있으므로 여기서 명령 수행함
        - 편의성을 위해 bastion 내부에 tinyproxy같은 포워딩 프록시를 띄워두고 로컬 kubectl에서 이 프록시 타도록 설정함
            - 터널링 (Proxy Mode):
            
            ```yaml
            # 내 컴퓨터의 8888 포트를 Bastion의 8880 (Tinyproxy) 포트와 연결
            gcloud compute ssh dra-p-seo-bastion -- -L 8888:localhost:8880 -N -q -f
            ```
            
            - 환경변수 설정 (내 로컬 컴퓨터):
            
            ```yaml
            export HTTPS_PROXY=http://localhost:8888
            ```
            
            - kubectl 실행:
                - 이제 내 로컬에서 치는 kubectl get pods 명령어가 8888 포트(Bastion)을 타고 들어가서 Private Master에 도달함
                - Bastion에 직접 SSH 접속해서 불편하게 작업안해도
- Redis (Memorystore) GUI 접속 (2.1 섹션에서 다룬 것 공식화)
    - 터널 생성:
    
    ```yaml
    #내 컴터 63790 포트 -> redis (10.20.0.9):6379 연결
    gcloud compute ssh bastion -- -L 63790:10.20.0.9:6379
    ```
    
    - 접속: redis desktop manager 등의 툴에서 localhost:63790으로 접속

---

# 3. 물리 인프라: 컴퓨팅 및 클러스터 (Physical Infra: Compute & GKE)

목표: GKE Standard 기반의 클러스터 스펙 상세 정의. Autopilot의 Managed 방식을 버리고, 비용 효율성 + 세밀제어의 Standard 환경을 구축합니다. 

### 3.1. GKE 클러스터 구성 (Cluster Configuration)

AS-IS vs TO-BE 비교 (migration context)

| 구분 | Legacy Autopilot (AS-IS) | New Standard (TO-BE) | Rationale |
| --- | --- | --- | --- |
| Topology | Regional(Dev/Prod 동일)
3개 존에 마스터 노드 분산
비용: 클러스터 관리비(약70불) | Dev: Zonal (…-a)
- 단일 존 구성, 관리비 무료
Prod: Regional(asia-northeast3)
- 고가용성유지. 관리비 70불 지불 | 비용 최적화 |
| Access | Public Cluster
- 제어 평면(Master)이 공인 IP 가짐
- 로컬에서 인증(gcloud auth login)하면 바로 접속 가능 | Private Cluster
- 제어평면이 PRivate IP만 가짐
- Bastion/Proxy 통해서만 접근 | 보안강화 |
| Maintenance | Unspecified(설정 없음)
- 구글이 임의의 시간에 업그레이드 수행 | Fixed Window
- 예를 들어 새벽 4시(KST)로 고정 | 안정성 (업무시간 노드 업그레이드로 인한 pod 재시작 방지) |

**3.1.1. 기본 설정 (Common)**

- Version Channel: REGULAR
    - 기존과 동일 채널을 사용해 버전 호환성 이슈 최소화
- Dataplane V2: Enabled
    - kube-proxy 대신 eBPF 기반 고성능 네트워크 사용해 처리량 높이기
- Maintenance Window: start_time = “19:00” (Terraform이 사용하는 UTC 기준, KST로는 새벽 4시)
- Security:
    - enable_shielded_nodes = true (Secure Boot)
    - Master Authorized Networks: Bastion이 위치한 mgmt 서브넷 CIDR (10.255.0.0/28)만 마스터 접근 허용

**3.1.2. 환경별 토폴로지 전략 (Topology Strategy)**

- Dev Cluster (dra-d-seo-gke)
    - Type: Zonal Cluster
    - Location: asia-northeast-a(or b or c)
    - 특징
        - Control Plane 관리비 무료
        - 워커 노드 간 데이터 전송 비용(Zone to Zone) 무료
        - 리스크: A존 데이터센터 전체 장애 시 Dev 아예 마비됨

**3.1.3. 네트워크 및 보안 설정 (Private Cluster)**

- Private Nodes: true (워커 노드에 Public IP 할당 금지하고 Cloud NAT를 사용합니다.)
- Private Endpoint: true (마스터 노드에 Public IP 할당 금지하고 Bastion을 이용합니다.)
- Master IPv4 CIDR: 172.16.0.0/28
    - 마스터 노드와 워커 노드 간 통신을 위해 구글이 관리하는 별도 대역 (VPC CIDR와 겹치지 않게 설정함)
- IP Allocation Policy (VPC-Native)
    - 2.1.2에서 정의한 Secondary Range 이름을 매핑합니다.
    - cluster_secondary_range_name = “gke-pods”
    - services_secondary_range_name = “gke-services”

### 3.2. 노드풀 및 오토스케일링 전략 (Node Pools & NAP)

이곳이야말로 비용 절감이 가장 극적으로 일어나는 부분으로, 특히 Spot Instance를 사용하면 큰 비용 절감이 가능합니다. 동시에, NAP(Node Auto-Provisioning)를 통해 Autopilot과 유사한 수준의 관리 편의성은 유지합니다.

**3.2.1. 전략 비교 AS-IS vs TO-BE**

| 구분 | AS-IS (legacy Autopilot) | TO-BE (new standard) | 핵심 차이 (Key Difference) |
| --- | --- | --- | --- |
| 스케일링 | Pod 중심
- 파드가 늘어나면 구글이 알아서 어딘가에 배치함 | Node 중심(NAP)
파드가 늘어나면 NAP가 자동으로 최적의 노드를 구매(Provisioning)해서 배치함 | 사용자 경험은 유사하나, 비용 통제권이 우리의 것이 됨 |
| 머신 타입 | 선택 불가
- 범용(General) 스펙 강제 사용
- CPU:RAM = 1:4 고정 | 동적 선택(Dynamic)
Standard vs Highmem 자동 선택 | CPU 낭비 제거.
(기존에 메모리 때문에 불필요하게 비싼 CPU 비용 지불 경험 다수) |
| GPU | 지원(비쌈) | Dev: Spot Enabled(spot=true)
Prod: Spot Disabled(spot=false) - 서울 리전 재고 부족 대비 | 필요할 때만생성, Spot 사용 시 70% 저렴함 |

**3.2.2. 노드풀 구성 상세 (Node Pool Specs)**

클러스터를 시스템(System), 일반(CPU), AI(GPU) 세 가지 영역으로 나누어 관리합니다.

- 시스템 노드 풀 (System Pool)
    - 역할: Ingress, CoreDNS, ArgoCD 등 클러스터 생존 필수 파드
    - 구성: e2-standard-2 (2 vCPU, 8GB) x 1~3대 (On-Demand).
    - Why: 절대 죽지 않는 안전지대
    - Taint 설정: CriticalAddonsOnly=true:NoSchedule
        - 이유: 이 Taint가 없으면, 일반 API 파드들이 어? 여기 자리 있네? 하고 시스템 노드에 들어와서 비싼 자리를 빼앗을 수 있어 오직 시스템 파드들만 들어오게 막아야 함
    - Autoscaling: min_node_count = 1, max_node_count = 3
- CPU 워커 (NAP - General)
    - 역할: 일반 백엔드 API, 워커, 프론트엔드(만약 vercel 탈출한다면)
    - 설정 프로필:
        - Machine Family: e2-standard(기본), e2-highmem(메모리 부족 시 자동 선택)
        - Spot: Enabled (spot=true)
    - Resource Limits (NAP Constraints):
        - Max vCPU: 100 (클러스터 전체 코어 수 상한 선)
        - Max Memory: 400GB
        - 이러한 설정이 없으면 오토스케일링 버그 시 노드 무한 생성되어 요금 폭탄 맞을 수 있음
    - 동작:
        - 평소에는 e2-standard로 싸게 돕니다.
        - 만약 어떤 파드가 “나 메모리 많이 필요해!” 하고 찡찡대면 NAP가 알아서 highmem노드를 사옴
- GPU 워커 (NAP - AI)
    - 역할: Skin Analysis 등 AI 모델 서빙
    - 설정 프로필:
        - GPU Type: nvidia-l4 or nvidia-tesla-t4 (가성비)
            - Note: A100 같은 고가 GPU가 생성되지 않도록 T4로 제한함
        - Machine Type: n1-standard-4 (GPU와 호환되는 타입)
        - Spot: Dev(Enabled)/Prod(Disabled) (이유: 서울 리전 GPU 재고 부족 대비 가용성 확보를 위해 Prod는 정가 사용)
        - 동작:
            - 평소 비용 0원 (GPU 파드가 없으면 노드 0개)
            - GPU 파드가 배포되면 수 분 내로 노드가 생성

**3.2.3. GPU 운영 및 배포 전략**

Standard에서는 GPU를 사용하려면 명시적인 선언이 필요합니다.

- 드라이버 설치 자동화
    - Terraform 설정 시 guest_accelerator 옵션을 켜면, 구글이 노드 생성 시 NVIDIA 드라이버를 자동으로 설치해 줍니다.
- 파드 배포 방법(개발자 가이드) GPU가 필요한 서비스 (예: dermacode-skin-analysis)의 values.yaml에 예를 들어 다음과 같은 내용을 추가합니다.

```bash
resources:
	limits:
		nvidia.com/gpu: 1 #나 지피유 1개 주세요!

# 공통 차트에서 자동 처리되지만 원리는 다음과 같음
nodeSelector:
	cloud.google.com/gke-accelerator: nvidia-l4
tolerations:
	- key: "nvidia.com/gpu"
		operator: "Exists"
		effect: "NoSchedule"
```

- 주의 사항 (Spot GPU)
    - GPU 인스턴스는 Spot 재고가 부족할 때가 종종 있음
    - Prod 환경은 재고 부족에 의한 서비스 다운을 막기 위해 처음부터 On-Demand를 사용하도록 설정함

**개념 설명: Taint와 Toleration

- Taint: 오염/출입 금지 표지판
    - 누가: Node가 가짐
    - 의미: Spot 노드는 태생적으로 불안정하므로 구글이 노드를 만들 때 “나는 언제 죽을지 모르는 Spot이야”라는 딱지를 붙여 놓음
    - 효과: 이 표지판이 붙은 노드에는 일반적인 파드(DB, 시스템 파드)는 절대 배치되지 않음(스케줄러가 거름) 덕분에 중요한 파드가 실수로  Spot에 들어가서 죽는 사고를 방지함
    - 실제 값: cloud.google.com/gke-spot=true:NoSchedule
- Toleration: 면역/출입증
    - 누가: Pod가 가짐
    - 의미: 우리 API 서버는 죽어도 다시 뜨면 그만이므로, 파드 설정(deployment.yaml)에 나는 Spot 노드에 들어가도 괜찮아요 (면역있어요)라는 출입증을 쥐여 줌
    - 효과: 스케줄러가 이 출입증을 확인하고 Taint가 붙은 Spot 노드에도 이 파드를 배치해 줌
    
    ```bash
    # deployment.yaml 예시
    spec:
      template:
        tolerations:
        # Spot 노드 출입 금지 표지판(Taint)이 있어도, 나는 들어가겠다는 선언
        - key: "cloud.google.com/gke-spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
          
    #참고: value: "true"의 의미
    # Equal(정밀 검사): Key와 Value가 토씨 하나 안틀리고 똑같아야 한다.
    # Exists(대충 검사): Value는 상관 없고, Key만 있으면 통과시켜라
    ```
    

**3.2.4. 리소스 최적화 (Sizing)**

복잡한 VPA(수직 확장) 대신, 모든 환경에 HPA(수평 확장)을 기본 적용해 운영 복잡도를 낮추고 안정성을 확보합니다.

- 초기 전략 (Safe Start):
    - 리소스(requests)를 너무 타이트하게 잡지 않고, 약간의 여유를 두어 OOM을 방지합니다. (cpu: 100m, mem: 256Mi) - 특별히 무거운 서비스가 아니라면 이 기본값을 그대로 사용함
    - HPA 표준화:
        - Dev/Prod 공통 적용: 메모리 사용량이 70%를 넘으면 파드 개수를 자동으로 늘림
        - 이점: 파드 재시작 없이 트래픽/메모리 급증에 대응할 수 있음
    - 사후 최적화:
        - 운영하다가 Grafana와 같은 모니터링 툴을 보고 메모리가 너무 남네 싶으면 그 때 requests 값을 줄여서 배포함.
- HPA 표준 정책 (HPA Standard policy)
    - Trigger: 메모리 사용량 70% 도달 시 확장
    - Min Replicas: 2 (Global Default)
        - Why?
            - Spot 안정성: Dev 환경에서도 spot 노드 회수 시 서비스 중단 방지
            - 개발자가 prod 배포 시 values.yaml 수정할 필요 없이 기본값 그대로 배포
            - 버그 조기 발견: 분산 환경(concurrency) 이슈를 미리 발견 가능
    - Max Replicas: 필수 설정. 메모리 누수나 무한 스케일링으로 인한 요금 폭탄 방지 (5~10)

**3.2.5. 공용 차트의 상속과 오버라이드**

우리가 만드는 공용 헬름 차트는 부모 클래스, 디폴트 설정값 역할을 합니다.

- DevOps의 역할: 대부분의 서비스는 CPU 100m, 메모리 128Mi 정도면 돌아가더라, 이걸 default로 둘게
- BE 개발자의 역할: 내껀 더필요한데? → 그 부분만 Override하면 됨

```bash
# service-user-api/values.yaml
image:
  tag: "v1.0.0"
# 끝. (자동으로 cpu 100m, memory 128Mi 적용됨)

# service-skin-analysis/values.yaml
image:
  tag: "v1.0.0"

# 여기만 내 맘대로 고치면 됨!
resources:
  requests:
    cpu: "2000m"   # 덮어쓰기!
    memory: "4Gi"  # 덮어쓰기!
    nvidia.com/gpu: 1

replicaCount: 2    # 이것도 내 맘대로!
```

### 3.3. Workload Identity 구성 (Auth Integration)

목표: JSON 키 파일 완전 제거 (Keyless) - Google Coud의 IAM(GSA)와 K8s의 Service Account(KSA)를 1:1로 결합하여 파드가 별도의 키 파일 없이도 안전하게 클라우드 리소스 (SQL, GCS, Secret Manager)에 접근하도록 합니다.

**Node Service Account 전략

- Node SA Scope: cloud-platform (광범위 권한) 대신 최소 권한만 부여함
    - logging.write, monitoring.write, trace.append (오직 로그랑 메트릭만 쓰게)
    - 핵심: 실제 권한은 노드(VM)가 아니라 Workload Identity(Pod)에게만 줌. 노드는 껍데기임

**3.3.1. 계정 매핑 전략 및 명명 규칙 (Mapping Standards)**

인증이 제대로 작동하려면 GCP쪽의 이름과 K8s 쪽의 이름이 정확하게 매핑되어야 합니다. 앞서 정의한 Naming Rule을 여기에 적용합니다.

- ID 정의 (Identity Definition)

| 구분 | 리소스 (Resource) | 명명 규칙 (Format) | 실제 예시 (Example) |
| --- | --- | --- | --- |
| GCP | GSA (Google Service Account) | {Short Name}-{Env} | gauserapi-d |
| K8s | KSA (K8s Service Account) | service-{Full Name} | service-gaia-user-api |
| K8s | Namespace | {Scope} | gaia |
- 바인딩 공식 (IAM Binding Formula)
    - Terraform에서 이 둘을 연결해주는 접착제
    - roles/iam.workloadIdentityUser를 생성할 때 사용하는 문자열 공식으로, 이 형식을 틀리면 인증 오류가 발생함
        - Format: serviceAccount:{PROJECT_ID}.svc.id.goog[{NAMESPACE}/{KSA_NAME}]
        - Example: serviceAccount:dr-aesthetics-dev.svc.id.goog[gaia/service-gaia-user-api]

**3.3.2. Terraform 구현 스펙 (Implementation Spec)**

DevOps (Terraform 모듈)는 아래 3단계 작업을 자동으로 수행해야 합니다.

- GSA 생성
    - Resource: google_service_account
    - ID: gauserapi-d
    - DisplayName: Workload Identity SA for gaia-user-api
- IAM 권한 부여 (GCP 리소스 접근권)
    - Resource: google_project_iam_member (또는 bucket/sql 별 iam)
    - Role: roles/cloudsql.client, roles/storage.objectViewer 등 필요 권한 할당
- Kubernetes 메니페스트 설정 (K8s Config)
    - Resource: google_service_account_iam_binding
    - Role: roles/iam.workloadIdentityUser
    - Members: 위 3.3.1의 바인딩 공식 사용

**3.3.3. Kubernetes 매니페스트 설정 (K8s Config)**

Terraform이 길을 닦아놓으면, 실제 파드가 배포될 때에 KSA에 Annotation을 달아서 GSA를 호출해야 함. 이 부분은 공통 Helm Chart에 구현됨.

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-gaia-user-api # KSA 이름
  namespace: gaia             # 네임스페이스
  annotations: # 나는 GCP의 gauserapi-d 권한을 빌려 쓸 거야라고 선언
    iam.gke.io/gcp-service-account: gauserapi-d@{project_id}.iam.gserviceaccount.com
```

3.3.4. 개발자 경험 (DX; how to use)

개발자는 더 이상 key.json을 관리하거나, 환경변수에 넣거나, (기존방식) kubectl로 하나하나 생성 후 바인딩 하지 않습니다.

- Google SDK 사용 시 (GCS 등): SDK가 자동으로 환경을 감지하고 인증합니다.
- DB 비밀번호 등 (Secret Manager): ESO가 Workload Identity를 사용해 비밀번호를 가져와 주므로, 개발자는 그냥 환경변수(DB_PASSWORD 등)를 읽기만 하면 됩니다.

---

# 4. 물리 인프라: 데이터 및 스토리지 (Physical Infra: Data & Storage)

목표: 상태가 있는 (Stateful) 자원의 관리 및 격리 전략

### 4.1. Cloud SQL 아키텍처 (Database Strategy)

인스턴스 구성 (Dev: Single, Prod: HA) 및 파라미터 튜닝

논리적 DB 분리 및 유저 권한 격리 자동화 패턴

### 4.2. Object Storage (GCS Strategy)

서비스별 버킷 격리 패턴 (for_each)

Lifecycle Management (보관 주기 및 삭제 정책)

### 4.3. Artifact Registry (Container Registry)

이미지 저장소 구조 및 Vulnerability Scanning 정책

---

# 5. 플랫폼 시스템: 배포 및 트래픽 (Platform: GitOps & Ingress)

목표: ArgoCD 및 인그레스 컨트롤러 상세 설정

### 5.1. GitOps 엔진 구성 (ArgoCD Setup)

ArgoCD 자체 배포 전략 (Helm)

App of Apps 패턴 구조 및 프로젝트 격리

### 5.2. 인그레스 및 엣지 라우팅 (Unified Ingress Strategy)

Nginx Ingress Controller: 외부 트래픽의 유일한 물리적 진입점 (L4 LoadBalancer 비용 절감)

Cert-Manager: SSL/TLS Termination 처리

### 5.3. 애플리케이션 게이트웨이 통합 (API Gateway Integration)

External Gateway(Custom Go): Nginx Ingress 뒷단에 배포하여 비즈니스 로직 및 인증 처리. 라우팅 경로는 Client → Nginx Ingress → Go Gateway → Services.
Internal Gateway(Traefik): 마이크로서비스 간(East-West) 통신 라우팅 규칙 정의. K8s Service Discovery와의 연동 전략

** 5.2 Nginx Ingress 가 호텔 정문 역할을 하며 트래픽을 받아 SSL을 풀고, External Gateway라는 프론트 데스크로 넘겨서 구체적인 업무(Go 로직)을 수행한 뒤, Internal Gateway 등을 통해 내부 직원 (마이크로서비스)끼리 소통하는 구조.

### 5.4. 시크릿 관리 시스템 (Secret Management)

GCP Secret Manager + External Secrets Operator 연동 상세

비밀번호 Rotation 전략

**5.5. API 라우팅 및 URL 전략 (API Routing Strategy)**
**목표:** 마이크로서비스의 물리적 분리(Decoupling)를 유지하면서, 클라이언트에게는 단일 진입점(Single Entry Point)을 제공하는 **통일된 URL 규칙**을 정의합니다.
**5.5.1. 라우팅 아키텍처 (Traffic Flow)**
트래픽은 **L7 LoadBalancer(Nginx)**가 도메인을 식별하여 **External Gateway**로 전달하고, **External Gateway**가 URL 경로(Path)를 분석하여 **개별 마이크로서비스**로 라우팅하는 2단계 구조를 가집니다.
• **1단계 (Ingress):** `api.dr-aesthetics.com` → **External Gateway** (단일 파이프)
• **2단계 (Gateway):** `/v1/user/...` → **User Service** (Path 기반 분기)
**5.5.2. URL 구조 및 규칙 (URL Convention)**
모든 API 요청은 RESTful Resource 기반의 **Prefix Routing** 전략을 따릅니다.
**포맷:** `https://{domain}/{version}/{topic}/{resource}`**구성 요소설명예시비고Domain**서비스의 진입점`api.dr-aesthetics.com`Nginx Ingress가 처리**Version**API의 메이저 버전`v1`, `v2`Breaking Change 격리용**Topic라우팅의 기준 키(Key)**`user`, `payment`, `skin`Naming Rule의 **Topic**과 1:1 매칭**Resource**구체적인 행위`login`, `history`개별 서비스 내부 로직
**5.5.3. Prefix Stripping 전략 (핵심)**
Gateway는 트래픽을 뒷단(Upstream)으로 넘길 때, **라우팅의 기준이 되었던 Prefix(`/v1/{topic}`)를 제거(Strip)하고 전달**합니다.
• **이유 (Rationale):**
    ◦ **Decoupling:** 개별 마이크로서비스는 자신이 `/v1/user` 하위에 있는지, `/api/auth` 하위에 있는지 알 필요가 없어야 합니다. 오로지 자신의 비즈니스 로직(`/login`)에만 집중합니다.
    ◦ **유연성:** 나중에 Gateway에서 경로를 `/v2/members`로 바꾸더라도, 마이크로서비스 코드를 수정할 필요가 없습니다.
**[라우팅 예시 테이블]클라이언트 요청 URL (Public)Gateway 라우팅 판단대상 서비스 (Target)마이크로서비스 수신 URL (Private)**`POST /v1/user/login`Prefix: `/v1/userservice-gaia-user-apiPOST /loginGET /v1/skin/analysis/123`Prefix: `/v1/skinservice-dermacode-skin-apiGET /analysis/123GET /v1/payment/history`Prefix: `/v1/paymentservice-finance-payment-apiGET /historyPOST /v1/mobile/home`Prefix: `/v1/mobileservice-patient-app-bffPOST /home`
**5.5.4. BFF (Backend For Frontend) 통합 전략**
BFF 또한 Gateway 입장에서는 하나의 **마이크로서비스**로 취급합니다.
• **BFF 패턴:** 모바일, 웹 등 클라이언트 특성에 맞는 데이터 조합(Aggregation)이 필요한 경우.
• **라우팅:** `/v1/mobile/...`, `/v1/web/...`과 같이 클라이언트 타입이나 목적을 **Topic**으로 삼아 라우팅합니다.
• **흐름:** Gateway(인증/RateLimit) → BFF(데이터 조합) → Domain Service(User/Payment 등)
**5.5.5. 기술 스펙 (Implementation Spec)**
• **Component:** `service-gaia-gateway-api` (Custom Go Application)
• **Framework:** Gin-gonic
• **Proxy Lib:** `net/http/httputil.NewSingleHostReverseProxy`
• **Middleware Order:**
    1. **Request ID:** 추적용 UUID 생성
    2. **Logger:** 요청 기록
    3. **Rate Limiter (Redis):** 과도한 트래픽 차단
    4. **Authenticator (JWT):** 토큰 검증 및 `X-User-Id` 헤더 주입
    5. **Router (Reverse Proxy):** Prefix Stripping & Forwarding

---

# 6. 플랫폼 시스템: 관제 및 운영 (Platform: Observability)

목표: 모니터링, 로깅, 알림 파이프라인 구축

### 6.1. 로깅 아키텍처 (Logging)

Promtail(수집) → Loki(저장) → Grafana(조회) 파이프라인

GCS를 백엔드로 활용한 장기 보관 전략

### 6.2. 모니터링 및 알림 (Metrics & Alerting)

Prometheus Operator 구성

핵심 메트릭 정의 (USE/RED Method) 및 AlertManager 규칙

### 6.3. 모니터링 및 에러 추적 (Metrics & Error)

Prometheus/Grafana 및 Sentry 연동 표준

---

# 7. 애플리케이션 딜리버리 (Application Delivery)

목표: 개발자의 사용성을 결정하는 ‘골든 패스’ 정의

### 7.1. 공통 Helm Chart 사양(Spec)

values.yaml 인터페이스 명세 (개발자가 작성할 필드 정의)
내장 기능 정의 (HPA, PDB, ServiceMonitor, Migration Job)

### 7.2. DB 마이그레이션 표준 (DB Migration)

Flyway/Alembic 도구 컨테이너화 및 Helm Hook(pre-install) 실행 정의

### 7.3. 미들웨어 연동 전략 (Middleware)

PgBouncer 사이드카 vs 서비스 모드 결정

Redis/RabbitMQ 의존성 관리

### 7.4. CI/CD 파이프라인 상세 (Github Actions)

Workflow 단계별 정의 (Build → Test → Push → GitOps Update)

Image Promotion 프로세스 (Dev → Prod 승인 절차)

---

# 8. 개발자 경험 및 마이그레이션 (DX & Migration)

목표: 실제 사용 방법 및 전환 계획

### 8.1. 로컬 개발 환경 (Telepresence)

설치 가이드 및 네트워크 인터셉트 전략

### 8.2. AI/ML 워크로드 (AI/ML Ops)

TFLite 모델 서빙 구조 및 GPU 노드풀 확장 계획

### 8.3. AI Agent 인터페이스 (MCP Architecture)

MCP Server 배포: 인프라 상태(K8s, Logs)를 조회하는 에이전트용 API 서버 구축

Read-Only 권한 설계: Workload Identity를 통해 조회 권한만 안전하게 부여하는 IAM 정책

### 8.4. 마이그레이션 실행 계획 (Migration Plan)

파일럿 서비스 선정

데이터 이관 전략 (Downtime 최소화)