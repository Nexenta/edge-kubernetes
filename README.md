# edge-kubernetes
NexentaEdge Kubernetes integration

If you have questions, join us on the [nexenta_edge slack](https://nexentaedge.slack.com), channel **#general**.

## Deploy a Nexenta Edge cluster within kubernetes

- **Highly available Scale-Out Software Defined I/O Platform :**  **File + Block + Object!**
- Highly Scalable, Add storage nodes without impacting the cluster, auto-tuning.
- Intended to be deployed on clean Baremetal infrastructure or into existing k8s deployments
	on Baremetal HW. In future adding support for  **AWS, GCE, Azure** as
	they improve networking interfaces.

To deploy the cluster, depending on your current infrastructure :
- [Day 1 - if necessary](#Day1)
- 	kubespray + k8s 
- [Day 2](#Day2)
-	setup networking bridges for contiv + some sanity checks
-	deploy Edge via Helm chart

*  [Requirements](#requirements)

Supported Linux distributions
===============
Ubuntu 16.04, 18.04 (experimental)
RHEL-7.3

Requirements
--------------
Many of the base requirements come as part of kubespray + k8s, however **Nexenta Edge** has
specific requirements on versions of the same packages 

Versions of supported components
--------------------------------

[kubespray](https://github.com/kubespray/kubespray-cli) v0.4.4+ <br>
[kubernetes](https://github.com/kubernetes/kubernetes/releases) v1.9.0 <br>
[etcd](https://github.com/coreos/etcd/releases) v3.2.4 <br>
[contiv](https://github.com/contiv/install/releases) v1.0.3+ <br>
[docker](https://www.docker.com/) v1.13 (see below)<br>

Note: kubernetes doesn't support newer docker versions and we've seen issues related to
IPv6 networking support with docker > 1.13.x.

Day1
===============
- Install Kubespray on deployment machine
	```bash
	pip2 install kubespray
	wget https://raw.githubusercontent.com/kubespray/kubespray-cli/master/src/kubespray/files/.kubespray.yml
	mv .kubespray.yml ~/.kubespray/
	```
- And its deps..
	```bash
	add-apt-repository ppa:ansible/ansible && apt-get update && apt-get upgrade -y
	apt-get install ansible
	```

- Overlayfs on the rootfs of your nodes?
	```bash
	echo "service docker stop" >> fix_devmapper.sh
	echo "sed -i -e 's/DOCKER_OPTS\=/DOCKER_OPTS\=\-s devicemapper/g' /etc/init.d/docker" >> fix_devmapper.sh
	echo "rm -rf /var/lib/docker" >> fix_devmapper.sh
	echo "mkdir -p /var/lib/docker" >> fix_devmapper.sh
	echo "mkdir -p /var/lib/docker/devicemapper/devicemapper" >> fix_devmapper.sh
	echo "dd if=/dev/zero of=/var/lib/docker/devicemapper/devicemapper/data bs=1G count=0 seek=8" >> fix_devmapper.sh
	echo "service docker start" >> fix_devmapper.sh
	```
	
- Setup password-less ssh, req by kubespray
	```bash
	for ip in $HOSTS; do
		ssh-copy-id -i ~/.ssh/id_rsa.pub root@${ip}
	done
	```
- Install the deps for Kubespray on nodes
	```bash
	for ip in $HOSTS; do
		ssh $ip "add-apt-repository ppa:ansible/ansible && apt-get update && apt-get upgrade -y"
		ssh $ip "apt-get install -y docker.io apt-transport-https ansible"
		scp fix_devmapper.sh $ip:/root/
		ssh $ip "source /root/fix_devmapper.sh"
	done
	```

- Prepare and run Kubespray
	```bash
	hosts_str=""
	nodenum=1
	for i in $HOSTS; do
		hosts_str+="node${nodenum}[ansible_ssh_host=${i}] "
		nodenum=$(($nodenum+1))
	done
	kubespray prepare --nodes ${hosts_str} --etcds node1 node2 node3 --masters node1 node2
	kubespray deploy -n contiv
	```
- Install Helm
	```bash
	curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
	helm init
	```
Day2
===============

[ TBD ]

