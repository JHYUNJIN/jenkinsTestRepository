pipeline {
    // 스테이지 별로 다른 거
    agent any // 노예 , 아무(any) 노예

    triggers { // 3분 주기로 파이프라인을 구동하겠다는 트리거 지정
        pollSCM('*/3 * * * *')
    }

    environment { // 파이프라인을 채울 환경변수 선언
      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId') // aws명령어를 자유롭게 작성하기 위한 key
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey') // aws명령어를 자유롭게 작성하기 위한 key
      AWS_DEFAULT_REGION = 'ap-northeast-2' // 서울지역으로 디폴트 지정
      HOME = '.' // Avoid npm root owned
    }

    stages { // 
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/JHYUNJIN/jenkinsTestRepository.git',
                    branch: 'main',
                    credentialsId: 'jenkinsGitTest'
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                success { // 성공 시 메세지 출력
                    echo 'Successfully Cloned Repository'
                }

                always { // 성공하던 실패하던 항상 보낼 메세지
                  echo "i tried..."
                }

                cleanup { // 모두 완료했다는 로그
                  echo "after all other post condition"
                }
            }
        }

        // stage('Only for production') { 
        //   when {
        //     branch 'production' // 브랜치가 production이고 
        //     environment name : 'APP_ENV', value:'prod' // APP_ENV 값이 prod 이면 이 스테이지를 실행하라는 구문
        //     anyOf {
        //       environment name : 'DEPLOY_TO', value:'production'
        //       environment name : 'DEPLOY_TO', value:'staging'
        //     }
        //   }
        // }
        
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') { // ☆핵심
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            dir ('./website'){ // website안에 있는 파일들을 s3에 올려줘 
                sh '''
                aws s3 sync ./ s3://jhyunjinbuckettest 
                '''
            } // aws 버킷을 만들어 이름을 지정해준다. jhyunjinbuckettest
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'

                  mail  to: 'jhyunjin1889@gmail.com',
                        subject: "Deploy Frontend Success",
                        body: "Successfully deployed frontend!"

              }

              failure {
                  echo 'I failed :('

                  mail  to: 'jhyunjin1889@gmail.com',
                        subject: "Failed Pipelinee",
                        body: "Something is wrong with deploy frontend"
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {
              docker {
                image 'node:latest'
              }
            }
            
            steps {
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
        
        stage('Bulid Backend') { // 스텝이 끝나고 나서 실행될 구문
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){ // 서버는 도커를 만들어서 배포해야한다., 도커를 젠킨스 노드에 직접 깔고 도커 컨테이너를 실행시켜놔야한다.
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post { // 빌드하다 실패하면 에러 출력하고 나머지 파이프라인 종료해줘
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        
        stage('Deploy Backend') { // 배포
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){ // docker rm -f $(docker ps -aq) 154 // 컨테이너 종료하는 코드, 배포를 하지 않은 상태에선 안에 아무것도 없기 때문에 지금은 필요업고 배포가 된 이후부터는 필요함
                sh '''
                
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'jhyunjin1889@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
