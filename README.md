# my_ci_cd_test
ci/cd 테스트

# CI 절차
    - github 사이트 이동 -> 현 프로젝트로 이동
    - Setting -> Security -> Securits and variable > Actions
    - new repository 버튼 클릭 
        - git에 노출되면 안되는 정보들
        - 환경변수 (노출되면 안되는 키 정보, 디비정보 등등)
        - aws 계정관리 iam에서 계정별로 access ID/KEY 발급받아서 등록 -> 외부에서 aws 엑세스 가능
        - action에 의해 .env등 설정 파일로 이관
            - 소스코드상에는 없다
        - 등록 (필요한만큼, 수시로)
            - 이름, 값

    - gitcation에서 작동할 내용 기술
        - deploy.yml
        - 위치
            - 프로젝트루트/.github/workflows/deploy.yml 
            - gitaction을 작동시키는 이벤트(명령, 트리거)는 서브 브런치일 경우 아래처럼 추가 가능 
                ```
                    pull_request:
                    branches:   [dev]
                ```

            - jobs > step > name => 검증할 내용들을 추가할 수 있다
                - package.json 추가
                    - "build" : "babel routes -d build"
                        - 개발된 소스를 표준으로 변환
                        - npm install --save-dev babel-cli
                            - npm run build

# CD 절차
    - step 1 
        - 빌드한 소스코드. 리소스. 기타 설정파일 => 압축(zip)

    - step 2
        - aws 접속
            - IAM 계정상 엑세스 키, 엑세스 ID가 gitaction 시크릿 변수에 등록되어 있어야한다
            - 계정생성 => 키/아이디 발급 => 등록
            - 분실 노출 금지
        - step 3
            - 압축(zip)한 리소스 => s3 업로드

        - step 4
            - AWS codeDeploy 서비스에 절차에 따라 배포 시작

        - AWS 세팅
            - S3 진입
                - 버킷 만들기 클릭
                    - pusan092-deploy-bucket
        - EC2에 역할 부여
            - IAM 진입 
                - 엑세스 관리 > 역할 > 역할생성
                    - AWS 서비스
                        - 사용사례 > EC2 > 다음
                        - 권한추가
                            - AmazonS3FullAccess 체크
                            - AWSCodeDeployFullAccess 체크
                            - 다음
                        - 이름 지정, 검토 및 생성
                            - 역할이름 : pusan_deploy
                            - 역할 생성 클릭
            - EC2에 위에서 만든 역할을 보안에 적용
                - 인스턴스 선택
                - 작업 > 보안 > IAM 역할수정
                - pusan_deploy선택
                - IAM 역할 업데이트 클릭  
        - CodeDeploy 역할 부여
            - IAM 진입 
                - 엑세스 관리 > 역할 > 역할생성
                    - AWS 서비스
                        - 사용사례 > CodeDeploy > 다음
                        - 권한추가
                            - 그냥 다음
                        - 이름 지정, 검토 및 생성
                            - 역할이름 : pusan-code-deploy
                            - 역할 생성 클릭
            - CodeDeploy 진입
                - 배포 > 애플리케이션 > 애플리케이션 생성
                    - 이름 : pusan-code-deploy
                    - 플랫폼 : EC2/온프레미스
                - 배포그룹 생성 
                    - 이름 : dev
                    - 서비스 역할 : pusan-code-deploy
                - 환경구성
                    - Amazon EC2 인스턴스
                        - 키 : name
                        - 값 : 인스턴스 지정 : aws-cloud9-node- ...
                - 로드벨런서  체크 > 품
                - 배포그룹생성 클릭

        - IAM 사용자 추가
            - 목적 : 엑세스키, ID발급
            - IAM > 엑세스관리 > 사용자 > 사용자 생성
                - 이름 : user-cicd
                - 다음
                - 권한옵션
                    - AmazonS3FullAccess 체크
                    - AWSCodeDeployFullAccess 체크
                    - 다음
                - 사용자 생성
                    - 해당 유저로 진입
                        - 액세스 키 만들기
                        - 기타 > 다음
                        - 태그
                            - access-cicd
                        - 액세스 키 AK....
                        - 비밀 키 ubqz....
                        - 위의 2개 값을 가지고 github에 환경변수 등록


# CD - EC2 에서 수행할 작업
    - sudo apt install awscli
    - sudo aws configure
        - ID, 시크릿 키, region, 형식(json) 입력
    - codeDeploy agent
        - wget https://aws-codedeploy-ap-northeast-2.s3.amazonaws.com/latest/install
        - chmod +x ./install
        - sudo apt-get install ruby
        - sudo ./install auto
        - sudo service codedeploy-agent status
            - active (running) : 에이전트 작동중
    - ec2 서버가 재가동했을때 자동으로 에이전트 가동되게 설정 (옵션)
        - sudo nano /etc/init.d/codedeploy-startup.sh
            - 편집
            ```
            #!/bin
            sudo service codedeploy-agent restart
            ```
            - 저장 후 nano 종료 (ctrl + x, y, 엔터) 
        - sudo chmod +x /etc/init.d/codedeploy-startup.sh

# CD-소스파일
    - appspec.yml 파일 생성(루트)
        - codeDeploy 배포하는 애플리케이션에 대한 사양정리


IAM 사용자 삭제
s3 버킷 비우기 후 all 삭제