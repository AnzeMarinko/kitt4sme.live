Cluster Bootstrap
-----------------
> One-off procedure to build & set up your KITT4SME mesh infra.

We're going to put together a single-node MicroK8s cluster to host
the KITT4SME platform. What we'll build is pretty much the same as
in the [Demo Cluster][demo], but we're swapping out Minikube for
[MicroK8s][mk8s] so we can still start small but then easily add
more nodes in the future.

We've got an Ubuntu VM at `kitt4sme.collab-cloud.eu` to host the
platform. (Ubuntu 20.04.1 LTS, GNU/Linux 5.4.0-42-generic x86_64,
8 CPUs, 16GB RAM/4GB swap, 120GB storage.) You can SSH into the box
with e.g.

```bash
$ ssh martel@kitt4sme.collab-cloud.eu
```

The instructions below tell you what to do to build the platform on
this box, but if you want to try this at home any Ubuntu box with the
same specs will do. If you have Multipass, you can spin up a 20.4
Ubuntu VM

```bash
$ multipass launch --name kitt4sme --cpus 4 --mem 8G --disk 40G 20.04
$ multipass shell kitt4sme
```

and then follow the instructions below. Keep in mind, depending on
what you'll do with your toy platform later, you might need more RAM
and storage.


### Building your own cluster

If you're a Kitt4sme dev wanting to deploy to `kitt4sme.collab-cloud.eu`,
skip this section. If you're still reading, then I guess you'd like
to build your own Kitt4sme instance on your own box.

The first step is to fork `kitt4sme.live` on GitHub so you can use
your fork as a GitOps source for building your cluster. Then in your
fork edit (via ssh, to enable pushing changes)

* `deployment/mesh-infra/_replacements_/custom-urls.yaml`

to enter suitable values for the following fields:

* `argocd.repo`: URL of your GitHub fork. This will make Argo CD (see
  below) source the cluster build instructions from your repo instead
  of `kitt4sme.live`.
* `argocd.webapp`: Base URL of Argo CD Web UI. Replace the host part
  to match your hostname or IP address. You only going to need this
  setting if you later configure Argo CD to do SSO through Keycloak.
* `argocd.sso`: OIDC config for SSO through Keycloak. Like the above
  setting, you'll only need this if you want to do SSO. Here too, the
  only thing to change is the hostname/IP address to match yours.

Change TODOs (locally) in:
[custom-urls.yaml](https://github.com/AnzeMarinko/kitt4sme.live/blob/main/deployment/mesh-infra/_replacements_/custom-urls.yaml)
[app.yaml](https://github.com/AnzeMarinko/kitt4sme.live/blob/main/deployment/mesh-infra/argocd/projects/base/app.yaml)

* repoURL should be changed to the new forked repo
* URL parts should be changed to that of the VM IP
* You get Multipass VM IP by: `multipass list`

Then edit

* `deployment/mesh-infra/kustomization.yaml`

to uncomment the section where it says:

> Comment back in the following to customise your own Kitt4sme live instance.

Now commit your changes and push upstream to your fork.


### Tools

We'll use [Nix][nix] to avoid polluting the Ubuntu box with extras.
Install with

```bash
$ sh <(wget -qO- https://nixos.org/nix/install)
```

The script should output a message like

> Installation finished!  To ensure that the necessary environment
> variables are set, either log in again, or type
> 
> . /home/ubuntu/.nix-profile/etc/profile.d/nix.sh

Your path to `nix.sh` will likely be different from the above, just
copy-paste and run what the script says.

##### Note
* Always install the latest Nix. At the moment the latest version
  is `2.4`. You might install a newer version when you read this,
  but that's fine too.

Enable Nix flakes

```bash
$ mkdir -p ~/.config/nix
$ echo 'experimental-features = nix-command flakes' >> ~/.config/nix/nix.conf
```

There's a basic Nix flake in our repo we'll use to drop into a Nix
shell with the specific versions of the tools we need to manage the
cluster — e.g. `istioctl 1.11.4`. Let's create a convenience script
to start a Nix shell with our flake:

```bash
$ git clone https://github.com/AnzeMarinko/kitt4sme.live
$ cd kitt4sme.live/nix
$ nix shell
```

Notice our tools will only be available inside the Nix shell and will
be gone on exiting the shell. In particular, they won't override any
existing tool installation on this box. No dependency hell. But still
inside the Nix shell, you'll get the right version of each tool.

Download right version of the kubeseal:

```bash
$ wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.5/kubeseal-0.17.5-linux-amd64.tar.gz
$ tar -xf kubeseal-0.17.5-linux-amd64.tar.gz
$ sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```


### Cluster orchestration

We'll use [MicroK8s][mk8s] as a cluster manager and orchestration.
(Read the [Cloud instance][arch.cloud] section of the architecture
document about it.)

Install MicroK8s (upstream Kubernetes 1.21)

```bash
$ sudo snap install microk8s --classic --channel=1.21/stable
```

Add yourself to the MicroK8s group to avoid having to `sudo` every
time your run a `microk8s` command

```bash
$ sudo usermod -a -G microk8s $(whoami)
$ newgrp microk8s
```

and then wait until MicroK8s is up and running

```bash
$ microk8s status --wait-ready
```

Finally bolt on DNS and local storage

```bash
$ microk8s enable dns storage
```

Wait until all the above extras show in the "enabled" list

```bash
$ microk8s status
```

##### Notes
- *Istio*. Don't install Istio as a MicroK8s add-on, since MicroK8s
  will install Istio 1.5, which is ancient!
- *Storage*. MicroK8s comes with its own storage provider
  (`microk8s.io/hostpath`) which the storage add-on enables
  as well as creating a default K8s storage class called
  `microk8s-hostpath`.


Now we've got to [broaden MicroK8s node port range][mk8s.port-range].
This is to make sure it'll be able to expose any K8s node port we're
going to use.

```bash
$ nano /var/snap/microk8s/current/args/kube-apiserver
# add this line
# --service-node-port-range=1-65535

$ microk8s stop
$ microk8s start
```

Since we're going to use vanilla cluster management tools instead of
MicroK8s wrappers, we've got to link up MicroK8s client config where
`kubectl` expects it to be:

```bash
$ mkdir -p ~/.kube
$ ln -s /var/snap/microk8s/current/credentials/client.config ~/.kube/config
```

Now if you drop into the Nix shell, you should be able to access the
cluster with plain `kubectl`. As a smoke test, try

```bash
$ nix shell
$ kubectl version
$ kubectl get all --all-namespaces
```

Don't exit the Nix shell as we'll need some of the tools for the rest
of the bootstrap procedure.


### Mesh infra

[Istio][istio] will be our mesh infra software. (If you're not sure
what that means, go read the [Cloud instance][arch.cloud] section of
the architecture document :-)

Deploy Istio to the cluster using our own profile

```bash
$ wget -q -O profile.yaml https://raw.githubusercontent.com/AnzeMarinko/kitt4sme.live/main/deployment/mesh-infra/istio/profile.yaml
$ istioctl install -y --verify -f profile.yaml
```

Platform infra services (e.g. FIWARE) as well as app services (e.g.
AI) will sit in K8s' `default` namespace, so tell Istio to auto-magically
add an Envoy sidecar to each service deployed to that namespace

```bash
$ kubectl label namespace default istio-injection=enabled
```

Have a read through the [Demo Cluster][demo] section on installing
Istio for more info about our Istio setup with add-ons.


### Continuous delivery

[Argo CD][argocd] will be our declarative continuous delivery engine.
(Read the [Cloud instance][arch.cloud] section of the architecture
document about our IaC approach to service delivery.) Except for the
things listed in this bootstrap procedure, we declare the cluster
state with YAML files that we keep in the `deployment` dir within
[our GitHub repo][kitt4sme.live]. Argo CD takes care of reconciling
the current cluster state with what we declared in the repo.

For that to happen, we've got to deploy Argo CD and tell it to use
the YAML in our repo to populate the cluster. Our repo also contains
the instructions for Argo CD to manage its own deployment state as
well as the rest of the KITT4SME platform — I know, it sounds like
a dog chasing its own tail, but it works. So we can just build the
YAML to deploy Argo CD and connect it to our repo like this (**replace
the GitHub repo URL with that of your fork** if you're building your
own Kitt4sme cluster)

```bash
$ kustomize build https://github.com/AnzeMarinko/kitt4sme.live/deployment/mesh-infra/argocd | kubectl apply -f -
```

After deploying itself to the cluster, Argo CD will populate it with
all the K8s resources we declared in our repo and so slowly the KITT4SME
platform instance will come into its own. This will take some time.
Go for coffee.

##### Note
* Argo CD project errors. If you see a message like the one below in
  the output, rerun the above command again — see [#42][boot.argo-app-issue]
  about it.
  > unable to recognize "STDIN": no matches for kind "AppProject" in version "argoproj.io/v1alpha1"


### Post-install steps

Run some smoke tests to make sure all the K8s resources got created,
all the services are up and running and there's no errors. The only
unhappy service should be Keycloak.

Run:

```bash
$ kubectl get pod --all-namespaces
```

All the services should be in the “running” stage after some 15-30min, except:

* profilers
* postgres
* quantumleap
* keycloak

The reason for that is we create the initial admin user through a
sealed secret which the Sealed Secrets controller can't unpack. In
fact, you've got to generate the initial secrets to use with the new
cluster as explained in the [security how-to][sec]. Among those secrets
there's the one for the Keycloak admin.

Another one of those secrets contains the Argo CD admin user password.
(The user name is automatically generated by Argo CD to be: `admin`.)
Notice Argo CD automatically generates an admin password on the first
deployment. To show it, run

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

ArgoCD should be working now:

* Go to VM_IP/argocd using web browser
* You should see the login page.
* You can actually login:
* username: admin
* password: see the code above -> copy paste the output

You can use it if you get in trouble during the bootstrap procedure,
but keeping it around is like an accident waiting to happen. So you
should definitely zap it as soon as you've managed to log into Argo
CD with the password you entered in our sealed secret. To do that,
just

```bash
$ kubectl -n argocd delete secret argocd-initial-admin-secret
```

Finally, if you like, you can set up remote access to the cluster
through `kubectl`. One quick way to do that is to

1. Copy the content of `~/.kube/config` on `kitt4sme.collab-cloud.eu`
   over to your box.
2. Edit the file to change `server: https://127.0.0.1:8081` to
   `server: https://kitt4sme.collab-cloud.eu:8081`.
3. Make it your current K8s config: `export KUBECONFIG=/where/you/saved/config`.

Or add an entry to your existing local K8s config if you have one.
Also, it's best to use the tools packed in our Nix shell on your
box too if you can. The procedure to install and use them is the
same as that detailed earlier for the cluster master node.

Now give yourself a pat on the shoulder. You've got a shiny, brand
new, fully functional KITT4SME cloud to...manage and maintain.
Godspeed!

Those services fail because they depend on some credentials that are passed as secrets.
We need to generate secrets for postgres and keycloak.

Generating secrets works like this:

You write plain text passwords in the templates, then run `kubeseal` on top of them, which generates the same
templates, but those passwords are now encrypted. Kubeseal connects to the cluster, takes its public key, and uses
it to encrypt the data. You then push those encrypted templates to the repo - GitOps (not the plain text ones!!!). ArgoCD recognizes
that there was a change in the repo. The `sealed-secret` service notices that new sealed secrets are available and
decrypts them and updates the kubernetes cluster with plain secrets that kubernetes knows how to handle. Each
service has then a “reloader” that notices that the plain secret changed and the new value needs to be used.

ArgoCD runs an update every couple of minutes. Not to wait for that, you can log in to ArgoCD and manually execute sync.

Generate secrets:

* `cd deployment/mesh-infra/security/secrets/`

change the password in the template:

* `nano templates/keycloak-builtin-admin.yaml`

then run:

* `kubeseal -o yaml < templates/keycloak-builtin-admin.yaml > keycloak-builtin-admin.yaml`

Repeat the same process for the postgres:

* `nano templates/postgres-users.yaml`
* `kubeseal -o yaml < templates/postgres-users.yaml > postgres-users.yaml`

Then:

* `git restore templates/*.yaml`
* git add new encrypted files and commit changes
* git push to repo

You can then either wait for the ArgoCD to pick up the changes and propagate them,
or manually trigger ArgoCD, by logging in to the ArgoCD and clicking on the
“sync apps” in the top left corner.
The status of all the services should be “running” after ~15min.

## Starting already deployed platform

```bash
multipass shell kitt4sme
cd kitt4sme.live/nix
nix shell
microk8s start
kubectl get pod --all-namespaces
```


[arch.cloud]: https://github.com/c0c0n3/kitt4sme/blob/master/arch/mesh/cloud.md
[argocd]: https://argoproj.github.io/cd/
[boot.argo-app-issue]: https://github.com/c0c0n3/kitt4sme.live/issues/42
[demo]: https://github.com/c0c0n3/kitt4sme/tree/master/poc
[istio]: https://istio.io/
[mk8s]: https://microk8s.io/
[mk8s.port-range]: https://github.com/ubuntu/microk8s/issues/284
[nix]: https://nixos.org/
[kitt4sme.live]: https://github.com/c0c0n3/kitt4sme.live
[sec]: ./security.md
