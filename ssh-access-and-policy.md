# SSH Access and Policy

This document describes how CF and Diego support SSH access to CF application instances.


## Developers: SSH Access to CF App Instances

### Diego-SSH CLI Plugin

The easiest way to start an SSH session into a CF app instance is to use the [SSH plugin for the CF CLI](https://github.com/cloudfoundry-incubator/diego-ssh/releases). For CF CLI v6.10.0+, you can install it from the CF-Community repo:

```
cf add-plugin-repo CF-Community http://plugins.cloudfoundry.org/
cf install-plugin Diego-SSH -r CF-Community
```

For older versions of the CF CLI, install the plugin via `cf install-plugin <binary-url>`, where `<binary-url>` is the URL from the [SSH plugin GitHub releases](https://github.com/cloudfoundry-incubator/diego-ssh/releases) appropriate for your OS and architecture.

If your app is running on Diego with SSH enabled and your operator has configured SSH access to be publicly routable, run

```
cf ssh <app-name>
```

to start an interactive SSH session in instance index 0 of your app. To target a different instance index with `cf ssh`, supply it via the `-i` flag. 

The `cf ssh` command also supports executing commands inside the app instance and forwarding local ports to it via the `-L` flag. Additionally, we plan to add support for a `cf scp` command to transfer files via the SCP protocol and for a command or flag to run the Diego-SSH plugin as a local SSH proxy to the instance.


### CF Authenticator access

It is also possible to connect to your SSH-enabled app instance via other SSH clients. If SSH is configured for your deployment, Cloud Controller's `/v2/info` endpoint contains an `app_ssh_endpoint` field with the domain name and port for the externally available SSH endpoint, and an `app_ssh_host_key_fingerprint` field with the fingerprint of the host key for the deployment. To connect to a particular instance of your app, run 

```
ssh -p <ssh-port> cf:<app-guid>/<instance-index>@<ssh-domain>
```

The password is any UAA-issued OAuth2 bearer token for a user authorized to modify that app, such as a user with the SpaceDeveloper role in the app's space. Such a token can be obtained from the CF CLI with the `cf oauth-token` command.

Transferring files with `scp` is invoked similarly, although the user name must be specified with the `-o` flag because of the `:` character present in it. For example, to copy a file out of the instance, run

```
ssh -P <ssh-port> -oUser=cf:<app-guid>/<instance-index> <ssh-domain>:<remote-file-to-retrieve> <local-file-destination>
```

More details on the CF Authenticator can be found in the [Diego-SSH README](https://github.com/cloudfoundry-incubator/diego-ssh/blob/master/README.md#cloud-foundry-via-cloud-controller-and-uaa).



## Developers and Managers: SSH Access Policy

### Cloud Controller Configuration

Cloud Controller provides several levels of granularity for enabling or disabling SSH access to CF app instances.

* Each CF app has an `enable_ssh` field that determines whether the Diego SSH server will be invoked alongside the main process in the app and whether users will be authorized to connect to it. Changing the value of `enable_ssh` triggers a new version of the app and hence restarts it on Diego with or without the SSH server running, as desired.

* Each CF space has an `allow_ssh` field that determines whether users are authorized to connect to the SSH server running inside their app instance.

* Cloud Controller has an `allow_app_ssh_access` configuration field that determines whether any user is authorized to connect to the SSH server running inside their app instance.

The Diego-SSH plugin allows users to inspect and alter the app and space fields through plugin commands:

* `cf enable-ssh <app-name>` and `cf disable-ssh <app-name>` set the value of `enable_ssh` on an app.
* `cf ssh-enabled <app-name>` reports whether SSH access is enabled for that app.
* `cf allow-space-ssh <space-name>` and `cf disallow-space-ssh <space-name>` set the value of `allow_ssh` on a space.
* `cf space-ssh-allowed <space-name>` reports whether SSH access is allowed for apps in that space.

The `allow_app_ssh_access` property must be changed through Cloud Controller's config file. For a BOSH-deployed CF, an operator can specify this config property via the `cc.allow_app_ssh_access` property described below.


### Restart Policy for Altered Instances

We understand that developers and operators alike may be concerned that unrestricted access to CF app instances can produce instances whose contents and processes differ from those that the platform has provisioned. Consequently, we propose a fairly stringent restart policy on CF app instances that have been accessed via SSH in ways that could reasonably alter the contents of the instance:

- Instances accessed via SSH port forwarding or source-mode SCP ('copying files out') will not be considered altered afterward and will not be restarted.
- Instances accessed by other SSH operations, such as an interactive session or sink-mode SCP ('copying files in') will be considered altered and will be restarted after the connection closes.

We intend to allow CC admins to configure this policy on a per-space basis, so that spaces intended solely for app development and experimentation can be exempt from this instance-restarting policy. These restrictions are not yet in place, but we expect to implement them before we consider the initial work on SSH access complete.


## Operators: Configuring Deployments for SSH Access to CF App Instances

### Diego configuration

The following BOSH properties are relevant for configuring SSH access to CF app instances in [diego-release](https://github.com/cloudfoundry-incubator/diego-release). These all pertain to configuring the SSH proxy instances that handle incoming SSH requests and connect them to app instances running on Diego.

- `diego.ssh_proxy.host_key`: A PEM-encoded RSA private key used by the Diego SSH proxy instances as their host key. Deployment operators should generate their own keypair for each deployment and supply that in the manifest.
- `diego.ssh_proxy.enable_cf_auth`: Whether to activate the CF Authenticator in the proxies. This should be enabled for the `cf ssh` plugin to be able to connect to the Diego SSH server running in the instance.
- `diego.ssh_proxy.enable_diego_auth`: Whether to activate the Diego Authenticator in the proxies. For a production CF+Diego deployment, this should be disabled, as it allows access to any instance running the Diego SSH server with only the receptor credentials. Development deployments may wish to enable it to provide an alternate mechanism to establish SSH sessions.

The properties `diego.ssh_proxy.cc.internal_service_hostname` and `diego.ssh_proxy.cc.external_port` should not require direct configuration if an operator uses the spiff-based manifest-generation scripts to generate the Diego deployment manifest.


### CF configuration

The following BOSH properties are relevant for SSH configuration in [cf-release](https://github.com/cloudfoundry/cf-release):

- `app_ssh.host_key_fingerprint`: Fingerprint of the public key presented by the SSH host (in this case, Diego's layer of SSH proxies). This should be the fingerprint of the public key of the keypair generated for the `diego.ssh_proxy.host_key` value in the Diego deployment manifest.
- `app_ssh.port`: Port of the externally routable SSH endpoint advertised through the Cloud Controller info endpoint.
- `cc.allow_app_ssh_access`: As mentioned above, whether to allow SSH access at all for CF app instances.

If SSH access is allowed for the CF deployment, Cloud Controller will advertise the SSH endpoint to be `ssh.<system-domain>`, accepting traffic on the port given in `app_ssh.port`.


### SSH Load Balancer configuration

If the HAproxy job from cf-release is used as the gorouter load balancer and `cc.allow_app_ssh_access` is set to true, HAproxy will also serve as the load balancer for Diego's SSH proxies. This configuration relies on the presence of the same consul server cluster that Diego components use for service discovery. This configuration also works well for deployments where all traffic on the system domain and its subdomains is directed towards the HAproxy job, as is the case for a BOSH-Lite CF deployment on the default `10.244.0.34.xip.io` domain.

For AWS deployments, where the infrastructure offers load-balancing as a service through ELBs, the deployment operator can provision an ELB to balance load across the Diego SSH proxy instances. This ELB should be configured to listen to TCP traffic on the port given in `app_ssh.port` and to send it to port 2222. In order to register the SSH proxies with this ELB, the ELB identifier should then be added to the `elbs` property in the `cloud_properties` hash of the Diego manifest's `access_zN` resource pools. If the spiff-based manifest-generation templates are used to produce the Diego manifest, these `cloud_properties` hashes should be specified in the `iaas_settings.resource_pool_cloud_properties` section of the `iaas-settings.yml` stub.
