##### clone ocp/k8s fork
```bash
https://github.com/openshift/kubernetes.git
cd kubernetes
```
##### add remote
```bash
git remote add fixstream https://github.com/swatisehgal/kubernetes.git
git fetch --all
```
##### cherry pick device-mngr fix
```bash
chrrey-pick ec94fe9a2a33432a557a622af061bfdd79938ba1
```
##### build kubelet binary
```bash
make all WHAT=cmd/kubelet GOFLAGS=-v
```
##### copy binary from \_output/bin/kubelet into the node. one way is injecting the workstation public key and then copy using scp
```bash
scp _output/bin/kubelet core@<sno_node_name>:/tmp/kubelet
```
##### ssh into the node
```bash
ssh core@<sno_node_name>
```
##### run as root
```bash
sudo -i
```
##### add the following file
```bash
mkdir /var/lib/kubelet/device-plugins/sample
touch /var/lib/kubelet/device-plugins/sample/registration
```
##### make node /usr path writeable
```bash
sudo mount -o remount,rw /usr
```
##### save old kubelet binary copy
```bash
cp /usr/bin/kubelet /usr/bin/kubelet.origin
```
##### load new kubelet binary
```bash
systemctl stop kubelet && cp /tmp/kubelet /usr/bin/kubelet && systemctl start kubelet
```
##### make change persistent across reboot
```bash
ostree admin unlock --hotfix
```
##### exit the node

##### outside the node lax kube-system ns podSecurity
```bash
oc get ns kube-system -o yaml | oc label -f - pod-security.kubernetes.io/enforce=privileged pod-security.kubernetes.io/audit=privileged pod-security.kubernetes.io/warn=privileged
```
##### deploy _device-plugin_ pod (manifest is in the repo)
```bash
oc apply -f sample-device-plugin-control-registration.yaml
```
##### deploy _testpod_ (manifest is in the repo)
```bash
oc apply -f devicerequest.yaml
```
##### reboot the node

##### with the fix in kubelet we expect pod to keep on pending after reboot until device registration. the deletion of the file triggers registration of the device plugin to Kubelet. if the file was never created, then the device plugin registration should never happen. it is the responsibility of the e2e test/ person testing manually to ensure that the file is created as part of the setup. after node boot up again we expect to see the _testpod_ pending.

##### ssh the node again and remove the file, which will resume the device plugin registration process
```bash
rm -f /var/lib/kubelet/device-plugins/sample/registration
```

##### now we expect to see the _testpod_ running.
