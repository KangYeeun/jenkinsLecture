pipeline {
    // 스테이지 별로 다른 거
    agent any

    triggers {  //파이프라인이 git에 몇 분 주기로 지정할 지
        pollSCM('*/3 * * * *')
    }

    environment { // 파이프라인 안에서 쓸 환경변수
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId')
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey')
      AWS_DEFAULT_REGION = 'ap-northeast-2'
      HOME = '.' // Avoid npm root owned
    }
    //AWS에 자유롭게 명령하기 위해
    //젠킨스 안의 시스템 환경변수에 들어감
    //CI 명령어 치면 AWS에 알아서 들어가서 조절함


// 파이프라인 짜는 코드

    stages {    // 큰 단계를 설정
       

        //stages 안에 stage 명령어 부분이 전체적인 레시피 순번 , 파이프라인 구성
       
        // 레포지토리를 다운로드 받음
        stage('Prepare') {  //1번
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/KangYeeun/jenkinsLecture.git',
                    branch: 'master',
                    credentialsId: 'tokenforjenkins'
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success {
                    echo 'Successfully Cloned Repository' //git을 pull 받았다
                }

                always {    //항상 보여주는 것
                  echo "i tried..."
                }


                //post 모두 끝나고 해주는 것
                cleanup {
                  echo "after all other post condition"
                }
            }
        }
        
        /*
        
        // 프로덕션일때만 쓴다
        stage('Only for production'){
          when{
            bransh 'production'
            environment name: 'APP_ENV', value: 'prod'
            anyOf{
              environment name: 'DEPLOY_TO', value: 'production'
              environment name: 'DEPLOY_TO', value 'staging'
            }
          }
        }
       */

       //2번째 단계
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            //AWS IAM을 크레덴셜에 설정해두었기에 가능
            dir ('./website'){    //이 파일을 모두 S3에 올릴거다
                sh '''
                aws s3 sync ./ s3://jenkinsyeeun
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'

                  mail  to: 'dockeryeeun@gmail.com',
                        subject: "Deploy Frontend Success",
                        body: "Successfully deployed frontend!"

              }

              failure {
                  echo 'I failed :('

                  mail  to: 'dockeryeeun@gmail.com',
                        subject: "Failed Pipelinee",
                        body: "Something is wrong with deploy frontend"
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {
              //도커에서 노드의 최신버전 이미지로 일 할 것이다
              //젠킨스가 이 레포지토리에서 받아와서 일 할 것임
              docker {
                image 'node:latest'
              }
            }
            
            steps {
              //node 서버,젠킨스에 노드가 없으니 도커에 노드 깔아서 도커에서 실행
              dir ('./server'){
                  sh '''
                  npm install&&
                  npm run lint
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'
            //서버는 도커를 만들어서 배포
            //도커 빌드 -> 젠킨스 노드에 도커를 깔아주고 도커 컨테이너를 실행해놔야함
            // 환경에 따라서 환경변수를 가져오게 변수로 설정해줌
            dir ('./server'){
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post {

            //보통 실패해도 다음 스테이지로 넘어감
            //실패하면 종료하라고 설정
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        //ecs나 쿠버네티스 업데이트 하는 곳 (원래는)
        //우리는 원래 있던 도커 도는 컨테이너 지우고, 새로운 도커 런
        stage('Deploy Backend') {
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh '''
                docker rm -f $(docker ps -aq)
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'dockeryeeun@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
