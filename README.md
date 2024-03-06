# Import images BOSH release

## Summary

import-images-release is a BOSH release that gives the ability to configure a manifest to specify images to pull onto instances.  When enabled, it ensures the import-images job is run to pull desired images to instances when:

1. instances are created during new cluster builds
2. instances are upgraded during cluster upgrades
3. instances are created to replace another

This release can be useful for air-gapped environments, e.g. environments that restrict access to the internet that therefore restrict pulling images from dockerhub, etc.  It is assumed in this scenario a private registry exists, is accessible, and contains the required images.  The registry can be secured with credentials (or not).  Retagging the image after placing on an instance is supported as well.

## Some resources for those unfamiliar with BOSH releases

- [BOSH release overview](https://bosh.io/docs/release/)
- Browse existing releases - [Cloud Foundry BOSH Releases](https://bosh.io/releases/) 
  - [os-conf-release](https://github.com/cloudfoundry/os-conf-release) is a particularly useful BOSH release that includes many jobs, notably `post-deploy-script`, `pre-start-script`, and `pre-stop-script`.  These are very useful, simple and quick to use jobs that come in handy for customizing BOSH deployments by injecting scripts into different phases of BOSH deployments.  Learn more about some of these phases on BOSH docs [Update Lifecycle](https://bosh.io/docs/job-lifecycle/) and subsequent Lifecycle Hooks section
- [Creating a Release](https://bosh.io/docs/create-release/)
- How to build the simplest possible BOSH release - [How to Create a Lean BOSH Release](https://www.cloudfoundry.org/blog/create-lean-bosh-release/)

## Getting started with import-images

In a safe place to experiment (sandbox or lab without active workloads is ideal), try the following:

1. find an existing cluster(s), or create a small test cluster
2. on a host with bosh director access, clone this project
3. inspect the import-images directory and files, with more focus on the spec and post-start.sh.erb files

### spec

The [spec](jobs/import-images/spec) file contains an example for how configuration must be supplied for the import-images job.  Defined in the config will be the image name, auth (credentials for private registry), and retagAs for cases when an image name must be changed.  For example, the below spec would be used if debian-base and nginx images were to be pulled, where the registry for debian-base requires credentials and the registry for nginx does not.  Also below, the debian image config includes retagAS, which optionally sets another tag after pulling:

```yaml
---
name: import-images

templates:
  post-start.sh.erb: bin/post-start

properties:
  images:
    description: |
      Define images to be pulled from registry. Each image should specify a `name`
      attribute.  Images can optionally provide `auth` if using a private registry.
      The value of `auth` must be base64 encoded user:password to the private 
      registry for that image.  Images can optionally also provide `retagAs` if
      it is desired to change an image's tag after pulling.
    example:
      images:
      - name: harbor.mydomain.com/debian-base:v2.0.0
        auth: dXNlcm5hbWU6cGFzc3dvcmQ=
        retagAs: registry.k8s.io/debian-base:v2.0.0
      - name: harbor.mydomain.com/nginx
        auth: dXNlcm5hbWU6cGFzc3dvcmQ=
```

### post-start.sh.erb

The [post-start.sh.erb](jobs/import-images/templates/post-start.sh.erb) file is a bash script with embedded ruby that performs the job actions.  In the content below you see name is required and must be supplied in the config, whereas auth (credentials for the registry) and retagAs are optional.  The job simply pulls an image using auth (if supplied).  If desired, it can be retagged after the pull with the value in retagAs.

```sh
#!/bin/bash

set -ex

<% p('images').each do | image_hash | %>
  <%  image_name = image_hash['name']
      image_auth = image_hash['auth']
      image_retagAs = image_hash['retagAs']

      if image_name.nil? || image_name.empty?
        raise "Image must be configured with a 'name' attribute. '#{image_hash}'"
      end
      %>

  # pull the image, using auth/credentials if supplied
  <% if image_auth.nil? || image_auth.empty? %>
      echo "Pulling <%=image_name%>"
      sudo /var/vcap/packages/containerd/bin/crictl --runtime-endpoint unix:///var/vcap/sys/run/containerd/containerd.sock pull <%= image_name %>
  <% else %>
      echo "Pulling <%=image_name%> using credentials" 
      sudo /var/vcap/packages/containerd/bin/crictl --runtime-endpoint unix:///var/vcap/sys/run/containerd/containerd.sock pull --auth <%=image_auth%> <%=image_name%>
  <% end %>

  # retag the image after it has been pulled from registry
  <% if image_retagAs.nil? || image_retagAs.empty? %>
      echo "No retagAs specified for <%= image_name %>, not retagging..."
  <% else %>
      echo "Retagging <%= image_name %> to <%= image_retagAs %>"
      sudo /var/vcap/packages/containerd/bin/ctr --namespace=k8s.io image tag <%= image_name %> <%= image_retagAs %>
  <% end %>

<% end %>
```

4. create a BOSH release for import-images:

```sh
bosh create-release --version=0.2.0 --final
```

A successful run results in something similar to the below:

```
Adding job 'import-images/00d34c61ffd30581ca777aba56b4aea6537a5522332525d59d475cbb2c91606d'...
-- Started uploading 'import-images/00d34c61ffd30581ca777aba56b4aea6537a5522332525d59d475cbb2c91606d' (sha1=sha256:5b3a752295f021fabfba83b0527ac7018160bc4ff3e7df900e713fef1dd894c5)
-- Finished uploading 'import-images/00d34c61ffd30581ca777aba56b4aea6537a5522332525d59d475cbb2c91606d' (sha1=sha256:5b3a752295f021fabfba83b0527ac7018160bc4ff3e7df900e713fef1dd894c5)
Added job 'import-images/00d34c61ffd30581ca777aba56b4aea6537a5522332525d59d475cbb2c91606d'

Added final release 'import-images/0.2.0'

Name         import-images  
Version      0.2.0  
Commit Hash  745d91b  

Job                                                                             Digest                                                                   Packages  
import-images/00d34c61ffd30581ca777aba56b4aea6537a5522332525d59d475cbb2c91606d  sha256:5b3a752295f021fabfba83b0527ac7018160bc4ff3e7df900e713fef1dd894c5  -  

1 jobs

Package  Digest  Dependencies  

0 packages

Succeeded
```

5. upload the release:

```sh
bosh upload-release 
```

This should result in something similar to:

```
Using environment '192.168.1.11' as client 'ops_manager'

[-----------------------------------------------------------------] 100.00%   0s
Task 2209

Task 2209 | 21:30:13 | Extracting release: Extracting release (00:00:00)
Task 2209 | 21:30:13 | Verifying manifest: Verifying manifest (00:00:00)
Task 2209 | 21:30:13 | Resolving package dependencies: Resolving package dependencies (00:00:00)
Task 2209 | 21:30:13 | Creating new jobs: import-images/82cc204c231e5a410d7d5e4c8f3912a0d0b6fbd717cbd458dc5cf92922b52c5f (00:00:00)
Task 2209 | 21:30:13 | Release has been created: import-images/0.2.0 (00:00:00)

Task 2209 Started  Thu Feb 29 21:30:13 UTC 2024
Task 2209 Finished Thu Feb 29 21:30:13 UTC 2024
Task 2209 Duration 00:00:00
Task 2209 done

Succeeded
```

6. Verify the release:

```sh
bosh releases
```

Note the import-images release is included with the proper version:

```
Using environment '192.168.1.11' as client 'ops_manager'

Name                    Version                  Commit Hash  
antrea                  1.13.1-build.1*          207b21a  
aws-cpi                 1.27.1-build.1*          3bf4720+  
backup-and-restore-sdk  1.18.110*                ce76e95c  
bosh-dns                1.36.11*                 ec2d188c  
bosh-metric-sink        0.3.35*                  0429723  
bpm                     1.2.12*                  539aaad  
import-images           0.2.0                    5c28308  
kubo                    1.18.0-build.25.1.18.x*  9eb9e3b  
kubo-service-adapter    1.18.0-build.85*         5e4fb908  
kubo-windows            1.18.0-build.11.1.18.x*  f23fda0  
nsx-cf-cni              4.1.2.0.22596735*        8434fe8+  
nsx-cf-cni-windows      4.1.2.0.22596735*        8434fe8+  
pks-api                 1.18.0-build.85*         5bae3e54  
pks-nsx-t               1.106.0*                 8e941fc  
pks-telemetry           2.0.0-build.655*         6b12103c  
pks-vrli                0.17.8*                  8344245  
pks-vrops               0.24.1*                  33309b9+  
pxc                     1.0.20*                  a67e36b9  
sink-resources-release  3.1.21*                  e012090+  
syslog                  12.2.1*                  e354b90  
system-metrics          3.0.4*                   a2dfd50  
uaa                     74.5.95*                 a93f549  
vsphere-cpi             1.27.0-build.2*          97f8024+  
vsphere-csi             3.1.1-build.1*           a776f21+  
wavefront-proxy         0.32.1*                  fbc9a93  
windows-gmsa            0.1.0-build.4*           b28b0f3  
windows-syslog          1.2.2                    e656a73  
windows-utilities       0.11.0*                  6466a01  

(*) Currently deployed
(+) Uncommitted changes

28 releases

Succeeded
```

7. Create a import-images-runtime.yaml file that includes the data required by the [spec](jobs/import-images/spec).  Include full image paths.  Include a base64 encoded user:pwd string as auth, if required.  If an image must be retagged after pulling onto instances after deployment, put the new tag in retagAs.


8. Then, update our bosh config to include the import-images job and property configuration:

```sh
bosh update-config --type runtime --name import_images_runtime ~/import-images-runtime.yaml
```

10. Verify the config

```sh
bosh configs
```

See that the import_images_runtime config is active for the environment:

```
ubuntu@opsmanager-3-0:~/git/import-images-release$ bosh configs
Using environment '192.168.1.11' as client 'ops_manager'

ID   Type     Name                                                   Team                                            Created At  
4*   cloud    default                                                -                                               2024-02-12 16:26:48 UTC  
6*   cloud    pivotal-container-service-17db720dfc2bc7086588         pivotal-container-service-17db720dfc2bc7086588  2024-02-12 16:44:59 UTC  
33*  cloud    service-instance_3344e507-4f29-4d4d-a467-1624dfb1cd1e  pivotal-container-service-17db720dfc2bc7086588  2024-03-01 12:41:28 UTC  
12*  cloud    service-instance_85057478-d48a-4a1c-a312-067eed512796  pivotal-container-service-17db720dfc2bc7086588  2024-02-15 18:42:30 UTC  
5*   cpi      default                                                -                                               2024-02-12 16:26:48 UTC  
3*   runtime  director_runtime                                       -                                               2024-02-12 16:26:48 UTC  
34*  runtime  import_images_runtime                                  -                                               2024-03-01 18:29:48 UTC  
1*   runtime  ops_manager_dns_runtime                                -                                               2024-02-12 16:26:47 UTC  
2*   runtime  ops_manager_system_metrics_runtime                     -                                               2024-02-12 16:26:47 UTC  

(*) Currently active
Only showing active configs. To see older versions use the --recent=10 option.

9 configs

Succeeded
```

11. (Optional) use bosh ssh to ssh a worker node in a deployment in this environment to look at state prior to running the release.  For example, check out /var/vcap/sys/log for existing logs for other jobs and validate no import-images job has ever run

12. Apply a change
  - for opsman users, one way is to go to opsman -> review pending changes -> select TKGI and ensure the ‘upgrade all clusters errand’ option is checked, and apply changes — this will modify all clusters
  - or, using bosh you can use bosh run-errand like:

```sh
bosh -d pivotal-container-service-17db720dfc2bc7086588 run-errand upgrade-all-service-instances
```

13. Next, bosh ssh to the worker node and inspect the /var/vcap/sys/log/import-images logs, as well as puruse crictl images on the node to validate desired images are now present. If you retag an image you should see something similar to:

```
worker/4fa48395-0686-4d5f-9d46-c0d3ccd5827d:~$ crictl images
IMAGE                                                                        TAG                                        IMAGE ID            SIZE
harbor.mydomain.com/debian-base                                              v2.0.0                                     9bd6154724425       21.1MB
registry.k8s.io/debian-base                                                  v2.0.0                                     9bd6154724425       21.1MB
```

9. finally, create a new test cluster and verify the import-images job has been applied by also validating the logs and crictl images on a worker node
