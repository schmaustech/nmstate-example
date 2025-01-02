### Install and Configure NMState Operator

There will be a need to configure network interfaces on the nodes that were not configured at initial cluster creation time and the NMState operator is designed for those use cases.   The first step is to create a custom resource file that contains the namespace, operator group and subscription.

~~~bash
$ cat <<EOF > nmstate-operator.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: openshift-nmstate
    name: openshift-nmstate
  name: openshift-nmstate
spec:
  finalizers:
  - kubernetes
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: NMState.v1.nmstate.io
  name: openshift-nmstate
  namespace: openshift-nmstate
spec:
  targetNamespaces:
  - openshift-nmstate
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/kubernetes-nmstate-operator.openshift-nmstate: ""
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
~~~

Then we can take the customer resource file and create it on the cluster.

~~~bash
$ oc create -f nmstate-operator.yaml 
namespace/openshift-nmstate created
operatorgroup.operators.coreos.com/openshift-nmstate created
subscription.operators.coreos.com/kubernetes-nmstate-operator created
~~~

Next we should validate the operator is up and running.

~~~bash
$ oc get pods -n openshift-nmstate
NAME                               READY   STATUS    RESTARTS   AGE
nmstate-operator-d587966c9-qkl5m   1/1     Running   0          43s
~~~

A nmstate instance is required so we will create a custom resource file for that.

~~~bash
$ cat <<EOF > nmstate-instance.yaml
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
EOF
~~~

Then we will create the instance on the cluster.

~~~bash
$ oc create -f nmstate-instance.yaml 
nmstate.nmstate.io/nmstate created
~~~

Finally we will validate the instance is running.

~~~bash
$ oc get pods -n openshift-nmstate
NAME                                      READY   STATUS    RESTARTS   AGE
nmstate-cert-manager-6dc78dc6bf-ds7kj     1/1     Running   0          17s
nmstate-console-plugin-5b7595c56c-tgzbw   1/1     Running   0          17s
nmstate-handler-lxkd5                     1/1     Running   0          17s
nmstate-operator-d587966c9-qkl5m          1/1     Running   0          3m27s
nmstate-webhook-54dbd47d9d-cvsf6          0/1     Running   0          17s
~~~

Next we can build a NodeNetworkConfigurationPolicy.  The example below will configure a static ipaddress on the ens8f0np0 interface on nvd-srv-32.

~~~bash
$ cat <<EOF > nncp-static-ip.yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ens8f0np0-policy 
spec:
  nodeSelector: 
    kubernetes.io/hostname: nvd-srv-32.nvidia.eng.rdu2.dc.redhat.com
  desiredState:
    interfaces:
    - name: ens8f0np0 
      description: Configuring ens8f0np0 on nvd-srv-32.nvidia.eng.rdu2.dc.redhat.com
      type: ethernet 
      state: up 
      ipv4:
        dhcp: false 
        address:
        - ip: 10.6.145.32
          prefix-length: 24
        enabled: true
EOF
~~~

Once we have the customer resource file we can create it on the cluster.

~~~bash
$ oc create -f nncp-static-ip.yaml 
nodenetworkconfigurationpolicy.nmstate.io/ens8f0np0-policy created

$ oc get nncp -A
NAME               STATUS      REASON
ens8f0np0-policy   Available   SuccessfullyConfigured
~~~

We can validate that the ipaddress is set by looking inside the node at the interface.

~~~bash
$ oc debug node/nvd-srv-32.nvidia.eng.rdu2.dc.redhat.com
Starting pod/nvd-srv-32nvidiaengrdu2dcredhatcom-debug-8mx6q ...
To use host binaries, run `chroot /host`
Pod IP: 10.6.135.11
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host

sh-5.1# ip address show dev ens8f0np0
96: ens8f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 58:a2:e1:e1:42:78 brd ff:ff:ff:ff:ff:ff
    altname enp160s0f0np0
    inet 10.6.145.32/24 brd 10.6.145.255 scope global noprefixroute ens8f0np0
       valid_lft forever preferred_lft forever
    inet6 fe80::c397:5afa:d618:e752/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
~~~

End of NMState operator configuration section
