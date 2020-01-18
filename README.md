# Kubernetes Cluster Provisioning Guide
A simple guide to provision a k8s cluster on el7.

## Part 1: Bring the cluster online
Unless otherwise specified, all commands are to be run on all nodes.

1. Set hostname
   - Update /etc/hostname
   - Add the following line to /etc/hosts on the initial control plane node:

            [host ip] [hostname] cluster-endpoint

   - Add the following lines to /etc/hosts on all other nodes:

            [host ip] [hostname] 
            [initial node ip] cluster-endpoint

2. Configure firewall
   - Control plane node:

            firewall-cmd --permanent --add-port=6443/tcp
            firewall-cmd --permanent --add-port=2379-2380/tcp
            firewall-cmd --permanent --add-port=10250-10252/tcp
            firewall-cmd --reload

   - Worker node:

            firewall-cmd --permanent --add-port=10250/tcp
            firewall-cmd --permanent --add-port=30000-32767/tcp
            firewall-cmd --reload
   - Dual-purpose node:

            firewall-cmd --permanent --add-port=6443/tcp
            firewall-cmd --permanent --add-port=2379-2380/tcp
            firewall-cmd --permanent --add-port=10250-10252/tcp
            firewall-cmd --permanent --add-port=30000-32767/tcp
            firewall-cmd --reload

3. Disable and remove any swap space
4. Add k8s repository

        cat <<EOF > /etc/yum.repos.d/kubernetes.repo
        [kubernetes]
        name=Kubernetes
        baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
        enabled=1
        gpgcheck=1
        repo_gpgcheck=1
        gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
        EOF

5. Disable SELinux

        setenforce 0
        sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

6. Install Docker and Kubernetes utilities

        yum install -y docker kubelet kubeadm kubectl --disableexcludes=kubernetes

7. Start and enable kubelet service

        systemctl enable --now docker kubelet

8. Ensure all traffic goes through iptables

        modprobe br_netfilter
        cat <<EOF > /etc/sysctl.d/k8s.conf
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        EOF
        sysctl --system

9. Verify connectivity and retrieve k8s images

        kubeadm config images pull

10. **On the initial control plane node:** Bring up cluster (--pod-network-cidr option is required by flannel)

        kubeadm init --control-plane-endpoint=cluster-endpoint --pod-network-cidr=10.244.0.0/16 --upload-certs

11. **On all other control plane nodes:** Join cluster (This command is from the output of step 10. The specific values will be different.)

        kubeadm join cluster-endpoint:6443 --token ibirdm.pbv61rijkj2v07kj --discovery-token-ca-cert-hash sha256:e326a1f3ca9985ff957b16b4cc48e5a1a93b26f15fd7487876da368e5ea669ca --control-plane --certificate-key c6c4ceae3eb0f89213f5a099572d761f852ad2b4ef8771fcd7c4bb5e0ac3bfc7
        
12. **On all worker-only nodes:** Join cluster (This command is from the output of step 10. The specific values will be different.)

        kubeadm join k8s-cluster-endpoint:6443 --token ibirdm.pbv61rijkj2v07kj --discovery-token-ca-cert-hash sha256:e326a1f3ca9985ff957b16b4cc48e5a1a93b26f15fd7487876da368e5ea669ca


13. Allow root to manage with kubectl (Run this command, then start a new shell for this to take effect)

        echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> .bashrc


## Part 2: Configuring external components
These commands need only be run on **one** node.

1. Install flannel

        kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml

2. Install metallb
   Modify `configmaps/metallb.configmap.yaml` with the IP addresses you want to use, then run:

        kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
        kubectl apply -f configmaps/metallb.configmap.yaml

3. Install and activate ingress-nginx

        kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.27.1/deploy/static/mandatory.yaml
        kubectl apply -f services/ingress-nginx.yaml
        
## Part 3: Test everything with a simple deployment

1. Apply the deployment configuration

        kubectl apply -f deployments/hello-node.deployment.yaml

2. Apply the service configuration

        kubectl apply -f services/hello-node.service.yaml

3. Apply the ingress configuration

        kubectl apply -f ingress/hello-node.ingress.yaml

