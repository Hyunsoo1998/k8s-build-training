# 🚀 k8s-build-training

Ansible을 이용한 Kubernetes 클러스터 구축 및 학습을 위한 기록용 레포지토리입니다.
<br>
추후 서비스를 구축 할 예정입니다. <br>
각 Node의 리소스 및 설정은 상황에 따라 달라질 수 있습니다.

---

## 🖥️ Environment

| 항목 | 내용 |
|------|------|
| OS | RHEL 9 |
| Hypervisor | VMware |
| Tool | Ansible, Kubernetes |

### 🧾 Node Resource

| Node         | CPU   | Memory | Disk  |
|--------------|--------|--------|--------|
| MasterNode1  | 2 core | 4 GB   | 40 GB |
| MasterNode2  | 2 core | 4 GB   | 40 GB |
| MasterNode3  | 2 core | 4 GB   | 40 GB |
| WorkerNode1  | 2 core | 8 GB   | 80 GB |
| WorkerNode2  | 2 core | 8 GB   | 80 GB |
| WorkerNode3  | 2 core | 8 GB   | 80 GB |

---


## 🧩 Node 구성 및 초기 설정

1. **Ansible 설치 (Master Node)**
   - dnf install ansible 명령어로 설치
    <img width="1100" height="800" alt="캡처" src="https://github.com/user-attachments/assets/cfc1dd01-7c4a-4049-be19-f86e68351c16" />


2. **Inventory 구성**
   - 각 Node의 hostname 및 IP 주소를 설정  
    
<img width="1100" height="800" alt="캡처" src="https://github.com/user-attachments/assets/8ec8b824-feec-409c-abd1-01f00629559d" />


3. **SSH Key 설정**
   
   - ansible all -m ping → Permission denied 발생  
   - RHEL 9의 기본 SSH 설정에서 root 로그인 비활성화됨
     
  <img width="1100" height="800" alt="image 2" src="https://github.com/user-attachments/assets/a7145c39-9aea-49cc-b766-03526433ea39" />

   

5. **SSH 설정 변경**
   - /etc/ssh/sshd_config 에 PermitRootLogin yes 추가  
   - 설정 적용 후 systemctl restart sshd

<img width="1100" height="800" alt="image 5" src="https://github.com/user-attachments/assets/a2586248-378d-4a70-b4a4-2090ba2f611c" />


6. **Ping 확인**
   - 모든 Node에 SSH 연결 성공 확인  
<img width="1100" height="800" alt="image 6" src="https://github.com/user-attachments/assets/63db57fa-94f3-4761-abdd-288923b6639a" />


---

## 🧹 Swap 비활성화

Kubernetes는 자원을 정확하게 제어하기 위해 Swap을 비활성화해야 합니다.

- Ansible Playbook: swapoff-book.yml
- /etc/fstab에서 Swap 제거

<img width="1100" height="800" alt="image 7" src="https://github.com/user-attachments/assets/d1c1166a-0981-4c93-b4cf-5b57b261e0ca" />

  
- Playbook 실행 후 확인 결과:
<img width="1100" height="800" alt="image 9" src="https://github.com/user-attachments/assets/79172caa-eb1b-43e0-9d5f-18a16777e89a" />



⚙️ Kubernetes 환경 구성
Ansible Playbook을 이용해 k8s 설치에 필요한 환경 구성 및 패키지를 자동 설치합니다.

k8s_require_install.yml

```yml
---
- name: Install k8s
  hosts: all
  become: yes
  tasks:
    - name: Install base packages
      dnf:
        name:
          - bash-completion
          - iproute-tc
        state: present

    - name: Enabled and start chronyd
      systemd:
        name: chronyd
        state: started
        enabled: yes

    - name: Disable SELinux
      selinux:
        state: disabled

    - name: Load kernel modules
      copy:
        dest: /etc/modules-load.d/k8s.conf
        content: |
          overlay
          br_netfilter

    - name: Apply kernel modules immediately
      shell: |
        modprobe overlay
        modprobe br_netfilter

    - name: Set sysctl parameters
      copy:
        dest: /etc/sysctl.d/k8s.conf
        content: |
          net.bridge.bridge-nf-call-iptables = 1
          net.ipv4.ip_forward = 1
          net.bridge.bridge-nf-call-ip6tables = 1

    - name: apply sysctl parameters
      copy:
        dest: /etc/yum.repos.d/kubernetes.repo
        content: |
          [kubernetes]
          name=Kubernetes
          baseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/
          enabled=1
          gpgcheck=1
          gpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key

    - name: install k8s packages
      dnf:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present

    - name: enable kubelet service
      systemd:
        name: kubelet
        enabled: yes
```

<br>

<img width="1100" height="800" alt="image 11" src="https://github.com/user-attachments/assets/5e8039bb-c78c-4a01-ba53-47906a486f0d" />

<img width="1100" height="800" alt="image 12" src="https://github.com/user-attachments/assets/cd340588-640c-4ea9-b259-c2845ae23686" />

---

현재 /etc/fstab에 iso를 mnt로 등록하지 않은 상태입니다.
<img width="1100" height="800" alt="automount 1" src="https://github.com/user-attachments/assets/5c2896a7-12ab-469d-b5f4-f5b5f289c7d6" />



<br>
playbook을 이용하여, 모든 Node에 대해서 한번에 iso를 /mnt로 mount를 진행하겠습니다.

<br>
<br>
mount-auto.yml

```yml

---
- name: mount auto iso
  hosts: all
  become: yes
  tasks:
    - name: mount ISO to /mnt
      mount:
        path: /mnt
        src: /dev/sr0
        fstype: iso9660
        opts: loop
        state: mounted

```



<br>

playbook 실행 결과

<img width="1100" height="800" alt="automoun3" src="https://github.com/user-attachments/assets/6ab0e802-d428-47d4-9a4e-88c8cf7c0b90" />
<br>
<img width="1100" height="800" alt="automount4" src="https://github.com/user-attachments/assets/eba4c18f-67a0-4a6b-996f-df321ad3e0f2" />



✅ 실행 결과 <br>
모든 Node에 (kubelet,kubeadm,kubectl) 설치 완료

다음 단계에서 kubeadm init을 통해 클러스터 구성 예정

---

📌 참고 사항 <br>
실습 환경에서는 보안을 위해 root SSH 허용을 임시로 설정했습니다.

추후 서비스를 구축 할 예정입니다.

운영 환경에서는 SSH 보안 설정을 강화할 필요가 있습니다.
