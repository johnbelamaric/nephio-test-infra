# Quick Start for GCE

1. Create a VM:

   ```bash
   $ gcloud compute instances create --machine-type e2-standard-8 \
                                     --boot-disk-size 200GB \
                                     --image-family=ubuntu-2204-lts \
                                     --image-project=ubuntu-os-cloud \
                                     --metadata=startup-script-url=https://raw.githubusercontent.com/nephio-project/test-infra/main/e2e/provision/gce_init.sh \
                                     nephio-r1-e2e
   ```

   There are some optional metadata values you can pass (add them as
   comma-delimited key=value pairs in the `--metadata` flag).

   - `nephio-run-e2e` defaults to `false` but you can set it to `true` to run
     the full e2e suite, instead of just setting up the sandbox.
   - `nephio-setup-type` defaults to `r1` but `one-summit` will use the workshop
     code instead of the R1 code. Results are not guaranteed with that.
   - `nephio-setup-debug` defaults to `false` but `true` will turn on verbose
     debugging.
   - `nephio-test-infra-repo` defaults to
     `https://github.com/nephio-project/test-infra.git` but you can set it to
     your repository when testing changes to these scripts, and your repo will
     then be pulled by the `gce_init.sh` instead.
   - `nephio-test-infra-branch` defaults to `main` but you can use it along with
     the repo value to choose a branch in the repo for testing.

2. If you want to watch the progress of the installation, give it about 30
   seconds to reach a network accessible state, and then ssh in and tail the
   startup script exection:

   Googlers (you also need to run `gcert`):
   ```bash
   $ gcloud compute ssh ubuntu@nephio-r1-e2e -- \
                    -o ProxyCommand='corp-ssh-helper %h %p' \
                    sudo journalctl -u google-startup-scripts.service --follow
   ```

   Everyone else:
   ```bash
   $ gcloud compute ssh ubuntu@nephio-r1-e2e -- \
                    sudo journalctl -u google-startup-scripts.service --follow
   ```

4. Once it's done, ssh in and port forward the port to the UI (7007) and to
   Gitea's HTTP interface, if you want to have that (3000):

   Googlers (you also need to run `gcert`):
   ```bash
   $ gcloud compute ssh ubuntu@nephio-r1-e2e -- \
                    -o ProxyCommand='corp-ssh-helper %h %p' \
                    -L 7007:localhost:7007 \
                    -L 3000:172.18.0.200:3000 \
                    kubectl port-forward --namespace=nephio-webui svc/nephio-webui 7007
   ```

   Everyone else:
   ```bash
   $ gcloud compute ssh ubuntu@nephio-r1-e2e -- \
                    -L 7007:localhost:7007 \
                    -L 3000:172.18.0.200:3000 \
                    kubectl port-forward --namespace=nephio-webui svc/nephio-webui 7007
   ```

   You can now navigate to
   [http://localhost:7007/config-as-data](http://localhost:7007/config-as-data) to
   browse the UI.

5. You probably want a second ssh window open to run `kubectl` commands, etc.,
without the port forwarding (which would fail if you try to open a second ssh
connection with that setting).

   Googlers:
   ```bash
   $ gcloud compute ssh ubuntu@nephio-r1-e2e -- -o ProxyCommand='corp-ssh-helper %h %p'
   ```

   Everyone else:
   ```bash
   $ gcloud compute ssh ubuntu@nephio-r1-e2e
   ```

6. From that session, you can deploy a fleet of five edge clusters with this:

   ```
   $ cat > edge-clusters.yaml <<EOF
   apiVersion: config.porch.kpt.dev/v1alpha2
   kind: PackageVariantSet
   metadata:
     name: edge-clusters
   spec:
     upstream:
       repo: nephio-example-packages
       package: nephio-workload-cluster
       revision: v5
     targets:
     - repositories:
       - name: mgmt
         packageNames:
         - edge01
         - edge02
         - edge03
         - edge04
       template:
         annotations:
           approval.nephio.org/policy: initial
   EOF
   $ kubectl apply -f edge-clusters.yaml
   ```

   You can observe the progress by looking at the UI, or by using `kubectl` to
   monitor the various package variants, package revisions, and kind clusters
   that get created.

7. Once a kind cluster comes up, you can access it by getting its kubeconfig
   from the cluster. For example:

   ```
   $ kubectl get secret edge01-kubeconfig -o jsonpath='{.data.value}' | base64 -d > edge01-kubeconfig
   $ kubectl --kubeconfig edge01-kubeconfig get ns
   NAME                           STATUS   AGE
   config-management-monitoring   Active   5m25s
   config-management-system       Active   5m25s
   default                        Active   5m46s
   kube-node-lease                Active   5m48s
   kube-public                    Active   5m48s
   kube-system                    Active   5m48s
   $
   ```
