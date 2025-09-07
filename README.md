# k8s

🚀 Kubernetes 3‑Node Cluster on Arch Linux (Controller + 2 Workers)


Networking: Cilium (kube‑proxy‑less, eBPF, direct routing)

Container Runtime: containerd

OS: Arch Linux


---

🔹 Prerequisites

- 3 machines (Arch Linux):
	- k8s-controller (control plane, IP: 10.0.0.2)

	- k8s-worker-beelink (worker, IP: 10.0.0.15)

	- k8s-worker-optiplex (worker, IP: 10.0.0.16)


- All nodes must have:
	- Internet access

	- yay installed (AUR helper)

	- neovim or vim



---

🔹 Step 1: Base Setup (All Nodes)

Update system

	sudo pacman -Syu

Install required packages

	yay -S containerd kubeadm kubelet kubectl cni-plugins crictl

Enable services

	sudo systemctl enable --now containerd
	sudo systemctl enable kubelet

Configure containerd

	sudo mkdir -p /etc/containerd
	containerd config default | sudo tee /etc/containerd/config.toml

Edit /etc/containerd/config.toml → set:


	[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
	  SystemdCgroup = true

Restart:


	sudo systemctl restart containerd

Disable swap

	sudo swapoff -a
	sudo sed -i '/swap/d' /etc/fstab

⚠️ On Arch, zram swap is often enabled. Disable it:


	sudo systemctl disable --now systemd-zram-setup@zram0.service
	sudo systemctl mask systemd-zram-setup@zram0.service
	sudo swapoff /dev/zram0
	echo 1 | sudo tee /sys/block/zram0/reset

Verify:


	cat /proc/swaps
	# should be empty


---

🔹 Step 2: Initialize Controller


On k8s-controller:


	sudo kubeadm init --pod-network-cidr=10.244.0.0/16

Set up kubectl for your user:


	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config


---

🔹 Step 3: Install Cilium (kube‑proxy‑less, eBPF)


On k8s-controller:


	curl -L --remote-name https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
	sudo tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin
	rm cilium-linux-amd64.tar.gz

Install Cilium in native routing mode (no VXLAN, no iptables, pure eBPF):


	cilium install \
	  --version v1.16.3 \
	  --set kubeProxyReplacement=true \
	  --set k8sServiceHost=10.0.0.2 \
	  --set k8sServicePort=6443 \
	  --set routingMode=native \
	  --set autoDirectNodeRoutes=true \
	  --set ipv4NativeRoutingCIDR=10.244.0.0/16 \
	  --set bpf.masquerade=true \
	  --set enableIPv4Masquerade=true \
	  --set ipam.mode=kubernetes

Verify:


	cilium status --wait
	kubectl get pods -n kube-system


---

🔹 Step 4: Join Workers


On each worker (beelink and optiplex), run the join command from kubeadm init.

If you need a new one:


	kubeadm token create --print-join-command

Example:


	sudo kubeadm join 10.0.0.2:6443 --token <token> \
	  --discovery-token-ca-cert-hash sha256:<hash>

⚠️ Make sure swap is disabled on workers too (cat /proc/swaps must be empty).


---

🔹 Step 5: Verify Cluster


On controller:


	kubectl get nodes -o wide

Expected:


	NAME                  STATUS   ROLES           AGE   VERSION   INTERNAL-IP
	k8s-controller        Ready    control-plane   Xm    v1.33.4   10.0.0.2
	k8s-worker-beelink    Ready    <none>          Xm    v1.33.4   10.0.0.15
	k8s-worker-optiplex   Ready    <none>          Xm    v1.33.4   10.0.0.16


---

🔹 Step 6: Test Networking


Create a test deployment + service + busybox pod:


	# test-nginx.yaml
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: nginx
	spec:
	  replicas: 3
	  selector:
	    matchLabels:
	      app: nginx
	  template:
	    metadata:
	      labels:
	        app: nginx
	    spec:
	      containers:
	      - name: nginx
	        image: nginx:1.25
	        ports:
	        - containerPort: 80
	---
	apiVersion: v1
	kind: Service
	metadata:
	  name: nginx
	spec:
	  selector:
	    app: nginx
	  ports:
	  - port: 80
	    targetPort: 80
	    protocol: TCP
	  type: ClusterIP
	---
	apiVersion: v1
	kind: Pod
	metadata:
	  name: busybox
	spec:
	  containers:
	  - name: busybox
	    image: busybox:1.36
	    command: ["sleep", "3600"]
	    imagePullPolicy: IfNotPresent
	  restartPolicy: Always

Apply:


	kubectl apply -f test-nginx.yaml

Test from busybox:


	kubectl exec -it busybox -- sh
	wget -qO- http://nginx

✅ You should see the nginx welcome page.


---

🔹 Step 7: Cleanup Test

	kubectl delete -f test-nginx.yaml


---

✅ Summary

- Installed containerd, kubeadm, kubelet, kubectl, cni-plugins, crictl

- Configured containerd with SystemdCgroup = true

- Disabled swap (including zram) on all nodes

- Initialized control plane with kubeadm init

- Installed Cilium in kube‑proxy‑less, eBPF, direct routing mode

- Joined 2 workers

- Verified all 3 nodes are Ready

- Tested pod‑to‑pod and service networking with nginx + busybox
