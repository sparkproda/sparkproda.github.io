---
layout: post
title:  "Integrating Vagrant, Ansible and Virtualbox for DevOps Tool "
date:   2021-02-15
categories:  vagrant ansible virtualbox devops CI/CD
---
Vagrant를 사용하여 Virtualbox를 생성하고, Vagrant에서 ansible를 실행하여 여러가지 Devops tool을  Docker 기반으로 구성합니다.

# Integrating Vagrant, Ansible and Virtualbox for DevOps Tool


![Devops_tool](https://sparkproda.github.io/assets/Install_flow_of_Devops_tool.png)


##  사전 설치 준비 사항

로컬 PC에 아래 사항이 설치되어 있어야 합니다.

- Vagrant
- Ansible
- VirtualBox

## Ansible 변수 암호화

Vagrant Vault를 사용하여 변수 파일을 암호화 합니다.
Vagrant 실행 시 암호를 물어보게 됩니다.
~~~
ansible-vault encrypt vars.yaml  ## 암호화

ansible-vault edit vars.yaml  ## 수정시
ansible-vault view vars.yaml  ## 확인
~~~


## Vagrant를 사용하여 VM 생성

아래와 같이 Vagrantfile을 생성합니다.
아래 코드는 Ansible playbook 실행하는 부분도 추가되어 있습니다.
~~~
IMAGE_NAME = "bento/ubuntu-18.04"
N = 1

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 8192
        v.cpus = 2
    end

    config.vm.box_check_update = false

    (1..N).each do |i|
        config.vm.define "devops-vm-#{i}" do |node|
            node.vm.box = IMAGE_NAME
            node.disksize.size = '60GB'
            node.vm.network "private_network", ip: "192.168.77.#{i + 10}"
            node.vm.hostname = "devops-vm-#{i}"
            node.vm.provision "ansible" do |ansible|
                ansible.playbook="playbook.yaml"
                ansible.ask_vault_pass = true
            end
        end
    end

end    
~~~

## DevOps Tool 설치를 위한 Ansible Playbook 생성

Docker 설치 부터, Mysql, Gitea, Nexus, Jenkins, Quay를 Docker Daemon으로 기동시킵니다.

젠킨스는 docker 관련된 부분을 빌드시 Docker daemon이 필요한 부분으로 Docker in Docker 방식으로 Dockerfile를 생성하여 새로운 젠킨스 도커 이미지를 생성합니다. 아래 2번 사항에 설명되어 있습니다.

### 01. Nexsus 설치 Ansible playbook

~~~
- name: Create a network
  docker_network:
    name: devops_network      
- name: nexus 디렉토리 생성
  file:
    path: /mnt/nexus_data
    state: directory
    owner: vagrant
    group: vagrant
    mode: "u=rwx,g=rx,o=rx"              

- name: nexus container
  docker_container:
    name: nexus
    image: sonatype/nexus3
    pull: yes
    state: started
    user: root
    restart: yes
    restart_policy: always
    networks:
    - name: devops_network    
    privileged: true
    volumes:
    - "/mnt/nexus_data:/nexus-data:rw"
    ports:
    - "5000:5000"
    - "8081:8081"   
~~~

### 02. Docker in jenkins_docker

젠킨스 도커이미지에 추가적으로 Docker 설치를 위해서 아래와 같이 Dockerfile를 추가합니다. 그리고 Ansible를 사용하여 도커이미지를 생성합니다.

~~~
FROM jenkins/jenkins:lts

USER root
# RUN git config --global http.sslverify false
RUN touch /etc/apt/apt.conf.d/99verify-peer.conf \
&& echo >>/etc/apt/apt.conf.d/99verify-peer.conf "Acquire { https::Verify-Peer false }"
RUN apt-get update
RUN apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
 RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -   
 RUN apt-key fingerprint 0EBFCD88
 RUN add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
 RUN apt-get update && apt-get install -y  docker-ce docker-ce-cli containerd.io  


RUN usermod -a -G docker jenkins

USER jenkins
~~~

도커 이미지 생성을 위한 Ansible playbook 코드는 아래와 같습니다.


~~~
- name: Copy Jenkins Dockerfile from local host to remote host
  copy:
    src: "dockerfiles/jenkins_inc_docker/Dockerfile"
    dest: /home/vagrant/Dockerfile
    owner: vagrant
    group: vagrant
    mode: "u=rw,g=r,o=r"  

- name: Build Jenkins images with docker-daemon
  community.general.docker_image:
     build:
       path: "/home/vagrant"
       pull: yes
     name: "jenkins_docker"
     tag: v1
     source: build
~~~     

### 03. Jenkins 설치 Ansible playbook

젠킨스 도커 이미지가 생성된 후 아래 Ansible코드를 사용하여 설치를 진행합니다. 그리고 로그인시 필요한 패스워드도 Ansible 코드를 통해서 출력되도록 했습니다.

~~~
- name: 젠킨스 디렉토리 생성
  file:
    path: /mnt/jenkins_data
    state: directory
    owner: vagrant
    group: vagrant
    mode: "u=rwx,g=rx,o=rx"    

- name: jenkins container
  docker_container:
    name: jenkins_docker
    image: jenkins_docker:v1
    state: started
    restart_policy: always
    networks:
    - name: devops_network    
    privileged: true
    volumes:
    - "/etc/timezone:/etc/timezone"
    - "/etc/localtime:/etc/localtime"
    - "/var/run/docker.sock:/var/run/docker.sock"
    - "/mnt/jenkins_data:/var/jenkins_home"
    ports:
    - "9898:8080"
    - "50000:50000"       

- name: Check password for jenkins
  shell: cat /mnt/jenkins_data/secrets/initialAdminPassword
  register: result

- name: Print result password
  debug:
    msg:
      - "{{ result.stdout }}"
~~~

### 04. Mysql DB 설치 Ansible playbook

Gitea, Quay는 DB가 필요합니다. 이에 아래와 같이 mysql DB를 설치합니다.

~~~
- name: DB 디렉토리 생성
  file:
    path: /mnt/mysql_data
    state: directory
    owner: vagrant
    group: vagrant
    mode: "u=rwx,g=rx,o=rx"  


- name: mariadb container
  docker_container:
    name: mariadb
    image: mysql:5.7
    pull: yes
    state: started
    restart_policy: always
    networks:
    - name: devops_network    
    privileged: true
    volumes:
    - "/mnt/mysql_data:/var/lib/mysql:rw"
    ports:
    - "3306:3306"
    env:
      MYSQL_DATABASE: "{{ MYSQL_DATABASE }}"
      MYSQL_USER: "{{ MYSQL_USER }}"  
      MYSQL_PASSWORD: "{{ MYSQL_PASSWORD }}"
      MYSQL_ROOT_PASSWORD: "{{ MYSQL_ROOT_PASSWORD }}"
~~~

### 05. Gitea 설치 Ansible playbook

DB생성 및 DB user를 생성합니다. 그리고 Gitea Docker 데몬을 실행시킵니다.

~~~

    - name: Create gitea DB via command line
      shell: |
        docker exec -i mariadb sh -c "mysql -e \"CREATE DATABASE IF NOT EXISTS {{ gitea_MYSQL_DATABASE }}; \" \
      -uroot -p'{{ MYSQL_ROOT_PASSWORD }}'"
     ignore_errors: true

    - name: Create gitea User via command line
      shell: |
        docker exec -i mariadb sh -c  \
        "mysql -e \"GRANT ALL PRIVILEGES ON {{ gitea_MYSQL_DATABASE }}.* TO {{ gitea_MYSQL_USER }}@'%' IDENTIFIED BY '{{ gitea_MYSQL_PASSWORD }}'; FLUSH PRIVILEGES; \" \
        -u root -p'{{ MYSQL_ROOT_PASSWORD }}' mysql"
      ignore_errors: true  

    - name: Gitea 디렉토리 생성
      file:
        path: /mnt/gitea_data
        state: directory
        owner: vagrant
        group: vagrant
        mode: "u=rwx,g=rx,o=rx"    

    - name: Gitea container
      docker_container:
        name: Gitea_docker
        image: gitea/gitea:1.13.2
        state: started
        restart_policy: always
        networks:
        - name: devops_network    
        privileged: true
        volumes:
        - "/etc/timezone:/etc/timezone"
        - "/etc/localtime:/etc/localtime"
        - "/mnt/gitea_data:/data"
        ports:
        - "3000:3000"
        - "2222:22"    
        env:
          MYSQL_DATABASE: "{{ gitea_MYSQL_DATABASE }}"
          MYSQL_USER: "{{ gitea_MYSQL_USER }}"  
          MYSQL_PASSWORD: "{{ gitea_MYSQL_PASSWORD }}"
          MYSQL_ROOT_PASSWORD: "{{ MYSQL_ROOT_PASSWORD }}"                

~~~
### 06. Quay 설치 Ansible playbook

Quay 설치 시 Mysql, Redis, quay 3가지 부분이 필요합니다. 이에 필요한 이미지가 준비되어 있어야 합니다. 여기에서는 이미지(Redis, Quay)를 받은 상태에서 진행한 부분으로 설명합니다.
Mysql DB는 기존 설치된 Mysql를 사용합니다.

아래 Ansible코드는 로컬에 이미 다운로드 받은 이미지를 tar파일로 저장한 후 진행한 부분입니다.
여기에서는 이미지를 제공하지는 않습니다. 참고 부탁드립니다.

~~~
- name: Create quay DB via command line
  shell: |
    docker exec -i mariadb sh -c "mysql -e \"CREATE DATABASE IF NOT EXISTS {{ quay_MYSQL_DATABASE }}; \" \
      -uroot -p'{{ MYSQL_ROOT_PASSWORD }}'"
  ignore_errors: true

- name: Create quay User via command line
  shell: |
    docker exec -i mariadb sh -c  \
     "mysql -e \"GRANT ALL PRIVILEGES ON {{ quay_MYSQL_DATABASE }}.* TO {{ quay_MYSQL_USER }}@'%' IDENTIFIED BY '{{ quay_MYSQL_PASSWORD }}'; FLUSH PRIVILEGES; \" \
      -u root -p'{{ MYSQL_ROOT_PASSWORD }}' mysql"
  ignore_errors: true


- name: copy docker imagefile from local host to remote host
  copy:
    src: "{{ item }}"
    dest: /home/vagrant/{{ item }}
    owner: vagrant
    group: vagrant
    mode: "u=rw,g=r,o=r"  
  with_items:
    - quay-2.9.3.tar
    - redis-forquay.tar

- name: load docker images
  shell: docker load -i "{{ item }}"
  with_items:
    - quay-2.9.3.tar
    - redis-forquay.tar

- name: Check local image list
  shell: "docker images | cut --delimiter=' ' --fields=1"
  register: result

- name: Print docker images
  debug:
    msg:
      - "{{ result.stdout.split('\n') }}"

- name: quay redis 디렉토리 생성
  file:
    path: /mnt/quay_redis_data
    state: directory
    owner: vagrant
    group: vagrant
    mode: "u=rwx,g=rw,o=rw"              

- name: Redis container
  docker_container:
    name: Redis_docker
    image: quay.io/quay/redis:latest
    state: started
    restart_policy: always
    networks:
    - name: devops_network    
    privileged: true
    volumes:
    - "/etc/timezone:/etc/timezone"
    - "/etc/localtime:/etc/localtime"
    - "/mnt/quay_redis_data:/var/lib/redis/data"
    ports:
    - "6379:6379"           

- name: quay 디렉토리 생성
  file:
    path: /mnt/quay_data/{{ item }}
    state: directory
    owner: vagrant
    group: vagrant
    mode: "u=rwx,g=rw,o=rw"  
  with_items:     
    - conf
    - storage

- name: Quay container
  docker_container:
    name: Quay_docker
    image: quay.io/coreos/quay:v2.9.3
    state: started
    restart_policy: always
    networks:
    - name: devops_network    
    privileged: true
    volumes:
    - "/etc/timezone:/etc/timezone"
    - "/etc/localtime:/etc/localtime"
    - "/mnt/quay_data/conf:/conf/stack"
    - "/mnt/quay_data/storage:/datastorage"        
    ports:
    - "443:443"
    - "8989:80"                  
~~~
