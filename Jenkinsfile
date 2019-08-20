pipeline {
  agent {
    label "master"
  }
  options {
    timeout(time: 2, unit: "HOURS")
    skipDefaultCheckout()
  }
  environment {
    CUSTOM_REGISTRY = "cloud-native-image-registry.westus.cloudapp.azure.com"
    CUSTOM_TAG = "${env.BUILD_TAG}-rejected"
    RUNC_VERSION="v1.0.0-rc8"
    CRIO_VERSION="v1.14.6"
    BUILDAH_VERSION="v1.10.0"
    GO_VERSION="1.12.8"
    GO_TAR="go${GO_VERSION}.linux-amd64.tar.gz"
    GOROOT="/usr/local/go"
    GOPATH="/tmp/go"
    PATH="${env.PATH}:/usr/local/go/bin:${GOPATH}/bin"
    REPO_NAME="intel-device-plugins-for-kubernetes"
    REPO_DIR="$GOPATH/src/github.com/intel/${REPO_NAME}"
  }
  stages {
    /*stage("Builds") {
      agent {
        label "xenial-intel-device-plugins"
      }
      stages {
        stage('Checkout') {
          steps {
            checkout scm
          }
        }
        stage("Get requirements") {
          parallel {
            stage("go") {
              steps {
                sh "curl -O https://dl.google.com/go/${GO_TAR}"
                sh "tar -xvf $GO_TAR"
                sh "sudo mv go $GOROOT"
                sh "mkdir -p $GOPATH/src/github.com/intel"
                sh "cp -rf ${env.WORKSPACE} $REPO_DIR"
                dir(path: "$REPO_DIR") {
                  sh "go get -v golang.org/x/lint/golint"
                  sh "go get -v golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow"
                  sh "go get -v github.com/fzipp/gocyclo"
                }
              }
            }
            stage("buildah") {
              steps {
                sh "sudo apt-get -y install e2fslibs-dev libfuse-dev libgpgme11-dev libdevmapper-dev libglib2.0-dev libprotobuf-dev"
                sh "mkdir -p ${GOPATH}/src/github.com/containers"
                dir(path: "${GOPATH}/src/github.com/containers") {
                  sh "git clone --single-branch --depth 1 -b $BUILDAH_VERSION https://github.com/containers/buildah"
                }
                dir(path: "${GOPATH}/src/github.com/containers/buildah") {
                  sh 'make buildah TAGS=""'
                  sh "sudo cp buildah /usr/local/bin"
                  sh "sudo mkdir -p /etc/containers"
                  sh '''echo '[registries.search]' > registries.conf'''
                  sh '''echo 'registries = ["docker.io"]' >> registries.conf'''
                  sh "sudo mv registries.conf /etc/containers/registries.conf"
                  sh "sudo curl https://raw.githubusercontent.com/kubernetes-sigs/cri-o/$CRIO_VERSION/test/policy.json -o /etc/containers/policy.json"
                  sh "sudo curl -L https://github.com/opencontainers/runc/releases/download/$RUNC_VERSION/runc.amd64 -o /usr/bin/runc"
                  sh "sudo chmod +x /usr/bin/runc"
                }
              }
            }
          }
        }
        stage("make vet, lint, cyclomatic") {
          parallel {
            stage("make lint") {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make lint"
                }
              }
            }
            stage("make format") {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make format"
                }
              }
            }
            stage("make vet") {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make vet"
                }
              }
            }
            stage("make cyclomatic-check") {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make cyclomatic-check"
                }
              }
            }
            stage("make test BUILDTAGS=kerneldrv") {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make test BUILDTAGS=kerneldrv"
                }
              }
            }
          }
        }
        stage('make images') {
          parallel {
            stage("make images with docker") {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make images"
                }
              }
            }
            stage("make images with buildah") {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make images BUILDER=buildah"
                }
              }
            }
          }
        }
        stage('make demos') {
          parallel {
            stage('make demos with docker') {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make demos"
                }
              }
            }
            stage('make demos with buildah') {
              steps {
                dir(path: "$REPO_DIR") {
                  sh "make demos BUILDER=buildah"
                }
              }
            }
          }
        }
        stage('make push images') {
          steps {
            withDockerRegistry([credentialsId: "57e4a8b2-ccf9-4da1-a787-76dd1aac8fd1", url: "https://${CUSTOM_REGISTRY}"]) {
              dir(path: "$REPO_DIR") {
                sh "make push-all"
              }
            }
          }
        }
      }
    }*/
    stage('K8s master && workers') {
      agent {
        label "clrvmk8s"
      }
      stages {
        stage('Checkout') {
          steps {
            checkout scm
          }
        }
        stage('Get master node requirements') {
          steps {
            sh '/bin/bash --login -c "git clone https://github.com/clearlinux/cloud-native-setup.git"'
            sh '/bin/bash --login -c "git clone https://github.com/kubernetes-sigs/node-feature-discovery.git"'
            sh '/bin/bash --login -c "./cloud-native-setup/clr-k8s-examples/setup_system.sh"'
            sh '/bin/bash --login -c "./cloud-native-setup/clr-k8s-examples/create_stack.sh init"'
          }
        }
        stage('Configure master node') {
          steps {
            sh 'mkdir -p $HOME/.kube'
            sh 'sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config'
            sh 'sudo chown $(id -u):$(id -g) $HOME/.kube/config'
            sh '/bin/bash --login -c "./cloud-native-setup/clr-k8s-examples/create_stack.sh cni"'
            sh 'kubectl rollout status deployment/coredns -n kube-system --timeout=5m'
            sh 'kubectl create -f ./node-feature-discovery/nfd-master.yaml.template'
            sh 'kubectl apply -f ./node-feature-discovery/nfd-daemonset-combined.yaml.template'
            sh 'kubectl rollout status daemonset/nfd -n node-feature-discovery --timeout=5m'
            sh 'kubectl describe node'
          }
        }
        stage('Expose config to workers') {
          steps {
            sh 'mkdir ./configs'
            sh 'cp -i $HOME/.kube/config ./configs/config'
            sh 'kubeadm token create --print-join-command | tee ./configs/join'
            stash name: "configs", includes: "configs/**/*"
          }
        }
        stage('K8s QAT Worker') {
          agent {
            label "centos7-vm-k8s-worker"
          }
          environment {
            KERNEL_VF_DRIVER="c6xxvf"
            DPDK_DRIVER="vfio-pci"
            MAX_NUM_DEVICES="10"
          }
          stages {
            stage('Checkout') {
              steps {
                checkout scm
                sh 'sudo lsmod | grep -i vfio_pci'
              }
            }
            stage('Join QAT node to master') {
              steps {
                unstash 'configs'
                sh 'sudo yum update -y && sudo yum install -y git yum-utils cri-o cri-tools kubeadm kubelet --disableexcludes=kubernetes'
                sh 'git clone https://github.com/clearlinux/cloud-native-setup.git'
                sh 'sed -i "/^upate_os_version/d" ./cloud-native-setup/clr-k8s-examples/setup_system.sh'
                sh 'sed -i "/^add_os_deps/d" ./cloud-native-setup/clr-k8s-examples/setup_system.sh'
                sh 'RUNNER=crio ./cloud-native-setup/clr-k8s-examples/setup_system.sh'
                sh 'sudo $(cat ./configs/join) --cri-socket=/var/run/crio/crio.sock'
                sh 'sudo mkdir -p $HOME/.kube'
                sh 'sudo cp -i ./configs/config $HOME/.kube/config'
                sh 'sudo chown $(id -u):$(id -g) $HOME/.kube/config'
                sh 'kubectl get nodes'
              }
            }
            stage('Pull QAT images') {
              steps {
                withCredentials([usernamePassword(credentialsId: '57e4a8b2-ccf9-4da1-a787-76dd1aac8fd1', passwordVariable: 'RPASS', usernameVariable: 'RUSER')]) {
                  sh 'sudo crictl -r /var/run/crio/crio.sock pull --creds "${RUSER}:${RPASS}" $CUSTOM_REGISTRY/intel-qat-plugin:$CUSTOM_TAG'
                  sh 'sudo crictl -r /var/run/crio/crio.sock pull --creds "${RUSER}:${RPASS}" $CUSTOM_REGISTRY/crypto-perf:$CUSTOM_TAG'
                  sh 'sudo crictl -r /var/run/crio/crio.sock images'
                }
              }
            }
            stage('Build QAT device plugin') {
              steps {
                sh "sudo cp -rf ${env.WORKSPACE} $REPO_DIR"
                dir(path: "$REPO_DIR") {
                  sh "make qat_plugin"
                }
              }
            }
            /*stage('Deploy QAT device plugin directly on the host') {
              steps {
                sh "sudo $GOPATH/src/github.com/intel/intel-device-plugins-for-kubernetes/cmd/qat_plugin/qat_plugin -dpdk-driver ${DPDK_DRIVER} -kernel-vf-drivers ${KERNEL_VF_DRIVER} -max-num-devices ${MAX_NUM_DEVICES} -debug"
              }
            }*/
          }
        }
      }
    }
  }
}
