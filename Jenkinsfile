@Library('github.com/releaseworks/jenkinslib') _
// node {
//   stage("List S3 buckets") {
//     withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
//         AWS("--region=eu-west-1 s3 ls")
//     }
//   }
// }


def builderImage
def productionImage
def ACCOUNT_REGISTRY_PREFIX
def GIT_COMMIT_HASH
def ECR

pipeline {
    agent any
    stages {
        stage('Checkout Source Code and Logging Into Registry') {
            steps {
                echo 'Logging Into the Private ECR Registry'
                // withEnv(["AWS_ACCESS_KEY_ID", "AWS_SECRET_ACCESS_KEY", "AWS_DEFAULT_REGION"]) {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'jenkins-aws', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        AWS("--region=us-east-2 s3 ls")
                    //    ECR = AWS("ecr get-login-password --region us-east-2")
                    }
                    script {
                        GIT_COMMIT_HASH = sh (script: "git log -n 1 --pretty=format:'%H'", returnStdout: true)
                        ACCOUNT_REGISTRY_PREFIX = "802697411312.dkr.ecr.us-east-2.amazonaws.com"
                        // sh 'aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 802697411312.dkr.ecr.us-east-2.amazonaws.com'
                        // docker login --username AWS --password ${ECR} 802697411312.dkr.ecr.us-east-2.amazonaws.com
                        docker.login("--username AWS --password ${ECR} 802697411312.dkr.ecr.us-east-2.amazonaws.com")
                    }
                // }
            }
        }

        stage('Make A Builder Image') {
            steps {
                echo 'Starting to build the project builder docker image'
                script {
                    echo "Start"
                        // sh 'docker build -t 802697411312.dkr.ecr.us-east-2.amazonaws.com/webapp-builder:latest -f ./Dockerfile.builder'
                        // sh 'docker push 802697411312.dkr.ecr.us-east-2.amazonaws.com/webapp-builder:auto'


                    echo "First Step"
                    builderImage = docker.build("${ACCOUNT_REGISTRY_PREFIX}/webapp-builder:${GIT_COMMIT_HASH}", "-f ./Dockerfile.builder .")
                    echo "Second Step"
                    builderImage.push()
                    echo "Third Step"
                    builderImage.push("${env.GIT_BRANCH}")
                    echo "Fourth Step"
                    builderImage.inside('-v $WORKSPACE:/output -u root') {
                        sh """
                           cd /output
                           lein uberjar
                        """
                    }
                }
            }
        }

        

        stage('Unit Tests') {
            steps {
                echo 'running unit tests in the builder image.'
                script {
                    builderImage.inside('-v $WORKSPACE:/output -u root') {
                    sh """
                       cd /output
                       lein test
                    """
                    }
                }
            }
        }

        stage('Build Production Image') {
            steps {
                echo 'Starting to build docker image'
                script {
                    productionImage = docker.build("${ACCOUNT_REGISTRY_PREFIX}/webapp:${GIT_COMMIT_HASH}")
                    productionImage.push()
                    productionImage.push("${env.GIT_BRANCH}")
                }
            }
        }

 
        stage('Deploy to Production fixed server') {
            when {
                branch 'release'
            }
            steps {
                echo 'Deploying release to production'
                script {
                    productionImage.push("deploy")
                    sh """
                       aws ec2 reboot-instances --region us-east-2 --instance-ids i-0100af54cff749aaa
                    """
                }
            }
        }


        // stage('Integration Tests') {
        //     when {
        //         branch 'master'
        //     }
        //     steps {
        //         echo 'Deploy to test environment and run integration tests'
        //         script {
        //             TEST_ALB_LISTENER_ARN="arn:aws:elasticloadbalancing:us-east-1:089778365617:listener/app/testing-website/3a4d20158ad2c734/49cb56d533c1772b"
        //             sh """
        //             ./run-stack.sh example-webapp-test ${TEST_ALB_LISTENER_ARN}
        //             """
        //         }
        //         echo 'Running tests on the integration test environment'
        //         script {
        //             sh """
        //                curl -v http://testing-website-1317230480.us-east-1.elb.amazonaws.com | grep '<title>Welcome to example-webapp</title>'
        //                if [ \$? -eq 0 ]
        //                then
        //                    echo tests pass
        //                else
        //                    echo tests failed
        //                    exit 1
        //                fi
        //             """
        //         }
        //     }
        // }

 
        stage('Deploy to Production') {
            when {
                branch 'master'
            }
            steps {
                script {
                    PRODUCTION_ALB_LISTENER_ARN="arn:aws:elasticloadbalancing:us-east-2:802697411312:listener/app/production-website/226df1a74dfb1724/e650a6428952b6a4"
                    sh """
                    ./run-stack.sh webapp-production ${PRODUCTION_ALB_LISTENER_ARN}
                    """
                }
            }
        }
    }
}
