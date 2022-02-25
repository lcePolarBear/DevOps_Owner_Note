# jenkins CI/CD 实例 mnc

## 提前准备

1. 首先创建 git 仓库，并实现免密登录
2. 注意 git 普通账户免密登录在 .ssh/ 路径下的权限是有要求的

## 创建 Jenkins 流水线

1. 实现 git 拉取步骤，最总要的就是能够通过 Jenkins 所在服务器的私钥成功ssh访问到 git 仓库的服务器
    
    ![https://docimg9.docs.qq.com/image/ADcLX8fiDCJiq_7TU9g-ZQ.png?w=839&h=577](https://docimg9.docs.qq.com/image/ADcLX8fiDCJiq_7TU9g-ZQ.png?w=839&h=577)
    

![https://docimg7.docs.qq.com/image/d20j5bgR1WehO4Ee3v7GDg.png?w=840&h=770](https://docimg7.docs.qq.com/image/d20j5bgR1WehO4Ee3v7GDg.png?w=840&h=770)

![https://docimg6.docs.qq.com/image/IbcFubcONAAh-Zd0qrJdWg.png?w=1280&h=644.0027797081307](https://docimg6.docs.qq.com/image/IbcFubcONAAh-Zd0qrJdWg.png?w=1280&h=644.0027797081307)

![https://docimg9.docs.qq.com/image/qm-HVYLZ5IqvoXBsuKCbiw.png?w=738&h=751](https://docimg9.docs.qq.com/image/qm-HVYLZ5IqvoXBsuKCbiw.png?w=738&h=751)

1. 开始编写流水线，其打包镜像、发布镜像、部署镜像的步骤可以使用 sh 脚本完成

```groovy
def registry = "192.168.102.242"
def project = "mnc"
def app_name = "content_receive"
def image_name = "${registry}/${project}/${app_name}:${BUILD_NUMBER}"

pipeline {
    agent any
    stages {
        stage('pull') {
            steps {
                echo 'git 生成的流水线脚本'
                git credentialsId: '5705f26a-8530-4b00-8173-d46f880c1877', url: 'git@192.168.102.241:/data/mnc.git'
            }
        }
        stage('Build') {
            steps {
                sh """
                    docker build -f Dockerfile-content_receive -t ${image_name} .
                    docker login -u admin -p 'admin@aq123' ${registry}
                    docker push ${image_name}
                """
            }
        }
        stage('Deploy') {
            steps {
                sh """
                    docker stop ${app_name}-1 ${app_name}-2
                    docker rm ${app_name}-1 ${app_name}-2
                    docker pull ${image_name}
                    docker run --name ${app_name}-1 -itd ${image_name}
                    docker run --name ${app_name}-2 -itd ${image_name}
                """
            }
        }
    }
}
```

1. 通过引入可变参数实现镜像 tag （变量 BUILD_NUMBER ）的控制

![https://docimg9.docs.qq.com/image/pja--IwDloaqmu60VtWK4A.png?w=949&h=913](https://docimg9.docs.qq.com/image/pja--IwDloaqmu60VtWK4A.png?w=949&h=913)

1. 进一步增加安全性，封装 harbor 的用户名密码

![https://docimg9.docs.qq.com/image/ADcLX8fiDCJiq_7TU9g-ZQ.png?w=839&h=577](https://docimg9.docs.qq.com/image/ADcLX8fiDCJiq_7TU9g-ZQ.png?w=839&h=577)

![https://docimg3.docs.qq.com/image/8HCZNhlK62KRWBKABte6Nw.png?w=844&h=727](https://docimg3.docs.qq.com/image/8HCZNhlK62KRWBKABte6Nw.png?w=844&h=727)

![https://docimg6.docs.qq.com/image/0079Nssp8PbTcBZTp6ivZQ.png?w=675&h=714](https://docimg6.docs.qq.com/image/0079Nssp8PbTcBZTp6ivZQ.png?w=675&h=714)

![https://docimg4.docs.qq.com/image/TAONe_AStAnle0rf4ZK5vA.png?w=600&h=668](https://docimg4.docs.qq.com/image/TAONe_AStAnle0rf4ZK5vA.png?w=600&h=668)

![https://docimg9.docs.qq.com/image/r9TpfibmbRcz2keDhM712g.png?w=712&h=770](https://docimg9.docs.qq.com/image/r9TpfibmbRcz2keDhM712g.png?w=712&h=770)

![https://docimg5.docs.qq.com/image/AIgTVeiNFOc0D-NQ-oKz4w.png?w=950&h=745](https://docimg5.docs.qq.com/image/AIgTVeiNFOc0D-NQ-oKz4w.png?w=950&h=745)

```groovy
def registry = "192.168.102.242"
def project = "mnc"
def app_name = "content_receive"
def image_name = "${registry}/${project}/${app_name}:${BUILD_NUMBER}"

pipeline {
    agent any
    stages {
        stage('pull') {
            steps {
                git credentialsId: '5705f26a-8530-4b00-8173-d46f880c1877', url: 'git@192.168.102.241:/data/mnc.git'
            }
        }
        stage('Build') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'fd293c01-c390-4997-a87d-7a8a334b8190', passwordVariable: 'p', usernameVariable: 'u')]) {
                sh """
                    docker build -f Dockerfile-content_receive -t ${image_name} .
                    docker login -u ${u} -p '${p}' ${registry}
                    docker push ${image_name}
                """
                }
            }
        }
        stage('Deploy') {
            steps {
                echo 'deploy...'
                sh """
                    docker stop ${app_name}-1 ${app_name}-2
                    docker rm ${app_name}-1 ${app_name}-2
                    docker pull ${image_name}
                    docker run --name ${app_name}-1 -itd ${image_name}
                    docker run --name ${app_name}-2 -itd ${image_name}
                """
            }
        }
    }
}
```