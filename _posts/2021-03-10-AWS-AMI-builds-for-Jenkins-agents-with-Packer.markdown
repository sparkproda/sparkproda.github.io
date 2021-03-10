---
layout: post
title:  "Automated AWS AMI builds for Jenkins agents with Packer"
date:   2021-03-10
categories:  aws ami packer jenkins
---

# Automated AWS AMI builds for Jenkins agents with Packer

AMI 이미지는 전체 EC2 인스턴스의 백업입니다. EBS 스냅 샷은 AMI 이미지가 생성 될 때 EC2 인스턴스에 연결된 개별 EBS 볼륨을 백업합니다. 그러므로 Packer를 통해서 AMI 생성시 EBS 스냅샷도 같이 생성됩니다.
Pakcer에서 빌드 타입이 amazon-ebs인경우, AMI생성시 소스 AMI에서 EC2 인스턴스를 프로비저닝 한 다음 해당 머신에서 추가적인 패키지를 Scripts와 Ansible Playbook를 통해서 설치 진행하고 AMI를 생성합니다. 완료되면 EC2 인스턴스는 종료됩니다.

여기에서는 Jenkins Agent로 Packer, Ansible를 사용하기 위해서 추가적으로 Packer docker image를 생성합니다. packer dockerfile은 packer, ansible, Python 구성을 포함하여 아래와 같이 Dockerfile을 작성했습니다. Packer version은 1.7.0 입니다.

![AMI build using Packer](https://sparkproda.github.io/assets/Automated_AWS_AMI_builds_for_Jenkins_agents_with_Packer.png)

~~~
FROM alpine:3.7

ENV PACKER_VERSION=1.7.0

RUN apk update \
    && apk add --update \
       ca-certificates \
       unzip \
       wget \
	   bash \
    && rm -rf /var/cache/apk/* \
    && wget -P /tmp/ https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip \
    && mkdir -p /opt/packer \
    && unzip /tmp/packer_${PACKER_VERSION}_linux_amd64.zip  \
	  && mv packer /usr/local/bin/packer \
    && mkdir -p /data

RUN apk add --update --no-cache docker openrc \
    && apk add --no-cache ansible \
    && rm -rf /tmp/*  \
    && rm -rf /var/cache/apk/*  
RUN rc-update add docker boot    

RUN adduser -S jenkins -G docker

RUN apk add --no-cache \
    python3 \
    python3-dev

RUN cd /usr/bin \
  && ln -sf python3 python \
  && ln -sf pip3 pip

VOLUME ["/data"]
WORKDIR /data

ENTRYPOINT [ "tail", "-f", "/dev/null" ]

CMD ["/bin/bash"]
~~~

에제 Packer template를 준비합니다.

## Step 1: Prep Packer template

~~~
{
    "variables": {
      "aws_access_key": "{{env `AWS_ACCESS_KEY`}}",
      "aws_secret_key": "{{env `AWS_SECRET_ACCESS_KEY`}}",
      "vpc_region": "",
      "vpc_id": "",
      "vpc_public_sn_id": "",
      "vpc_public_sg_id": "",
      "instance_type": "t2.micro",
      "ssh_username": "ubuntu"
    },
    "builders": [
      {
        "type": "amazon-ebs",
        "access_key": "{{ user `aws_access_key` }}",
        "secret_key": "{{ user `aws_secret_key` }}",
        "region": "ap-northeast-2",
        "vpc_id": "",
        "associate_public_ip_address": true,
        "security_group_id": "",
        "source_ami_filter": {
          "filters": {
          "virtualization-type": "hvm",
          "architecture": "x86_64",
          "name": "ubuntu/images/hvm-ssd/ubuntu-bionic-18.04-amd64-server-*",
          "block-device-mapping.volume-type": "gp2",
          "root-device-type": "ebs"
          },
          "owners": ["099720109477"],
          "most_recent": true
        },  
        "instance_type": "t2.micro",
        "ssh_username": "ubuntu",
        "ami_name": "base-ubuntu-18.04-aws-base-{{timestamp}}",
        "ami_groups": "all",
        "tags": {
          "Name": "sparkproda",
          "TEAM": "sparkproda"
        },  
        "launch_block_device_mappings": [
          {
            "device_name": "/dev/sda1",
            "volume_type": "gp2",
            "volume_size": "30",
            "delete_on_termination": true
          }
        ]
      }
    ],
    "provisioners": [    
      {
        "type": "shell",
        "script": "install-package.sh"
      },
      {
        "type": "ansible",
        "playbook_file": "ansible_common.yml",
        "extra_arguments": [ "-vvvv" ],
        "ansible_env_vars": ["ANSIBLE_HOST_KEY_CHECKING=False"],
        "user": "ubuntu"
      }
    ],
    "post-processors": [
      {
        "type": "manifest",
        "output": "manifest.json",
        "strip_path": true
      }
    ]
  }
~~~

## Step 2: Prep script

Ansible 연동을 위해서 Python이 설치되어야 하므로 아래와 같이 설치를 스크립트 방식으로 설치진행합니다.

~~~
#!/bin/bash

sudo apt-get update
sudo apt-get install software-properties-common -y
sudo apt-add-repository ppa:deadsnakes/ppa -y
sudo apt-get update
sudo apt-get install python -y
sudo apt-get install python3.8 -y
~~~
## Step 3: Prep Ansible playbook

설치할 패키지를 Ansible를 통해서 설치합니다. 여기에서는 Docker를 설치 진행 했습니다.

그리고나서, 젠킨스를 사용하여 빌드합니다. 빌드할 때 agent는 사전에 생성한 packer docker image를 사용합니다.

## Step 4: Prep Jenkinsfile
젠킨스를 통해서 빌드하기 위해서 Jenknsfile을 생성합니다. 그리고,
AWS 접근 정보는 Jenkins에서 설정했습니다. 그리고 Jenkins agent에서 DNS lookup이 잘되도록 dns를 명시했습니다. 그리고 Jenkins Agent 인 Packer image는 별도 Dockerfile를 생성하여 만들었습니다.

~~~
pipeline {
  agent any
  environment {
    AWS_ACCESS_KEY     = credentials('jenkins-aws-secret-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
  }
  stages {
    stage ('Build')  {
      agent {
        docker {
          image 'packer_spark:v1'
          args '--dns 8.8.8.8'
        }
      }
      steps {
        sh 'whoami'
        sh 'packer validate BaseAmi.json'
        sh 'packer build BaseAmi.json'

      }
    }
  }
}
~~~
