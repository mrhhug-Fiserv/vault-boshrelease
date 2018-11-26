# vault-boshrelease

This is my first time attempt at a bosh deploy, so I'll be a bit more verbose, and also cover deploying the service broker. Most of the infomation I learned from this guide: https://bosh.io/docs/create-release/. And also from referencing the bosh deployed consul

After a simple clone, the directory structure will look like the following:
```
# tree .
.
|-- README.md
|-- blobs
|   |-- consul_binary
|   `-- vault_binary
|-- config
|   |-- blobs.yml
|   `-- final.yml
|-- fiserv-hashicorp-enterprise-vault-consul.yml
|-- jobs
|   |-- all-vaults-are-unsealed
|   |   |-- monit
|   |   |-- spec
|   |   `-- templates
|   |       `-- run
|   |-- consul_client
|   |   |-- monit
|   |   |-- spec
|   |   `-- templates
|   |       |-- bin
|   |       |   |-- consul_client_ctl
|   |       |   `-- monit_debugger
|   |       `-- config
|   |           `-- consul_client.json.erb
|   |-- consul_server
|   |   |-- monit
|   |   |-- spec
|   |   `-- templates
|   |       |-- bin
|   |       |   |-- consul_server_ctl
|   |       |   `-- monit_debugger
|   |       `-- config
|   |           `-- consul_server.json.erb
|   |-- smoke-tests
|   |   |-- monit
|   |   |-- spec
|   |   `-- templates
|   |       `-- run
|   |-- vault
|   |   |-- monit
|   |   |-- spec
|   |   `-- templates
|   |       |-- bin
|   |       |   |-- monit_debugger
|   |       |   `-- vault_ctl
|   |       `-- config
|   |           `-- vault.hcl.erb
|   `-- vault-init
|       |-- monit
|       |-- spec
|       `-- templates
|           `-- run
|-- packages
|   |-- consul
|   |   |-- packaging
|   |   `-- spec
|   `-- vault
|       |-- packaging
|       `-- spec
`-- src

18 directories, 23 files
```
The initial skeleton can be created by running `bosh init-release --git --dir <release_name>` I suggest you use the --git flag even if you currently have no intention of using git, because its such a pain to do it later, the .gitignore is so valuble. 
* README.md
  * This file
* config
  * Directory for the configuration of the release, most notably we will be storing metadata about our blobs
* blobs.yml
  * You should not have to alter the blobs.yml by hand, if there is anything in that file: you can just clear it out. You should use `bosh add-blob <path_to_blob_on_local_system> <package_name>` and bosh will calculate the metadata
* final.yml
  * Don't think this file does anything unless you do a final release - at the time of writing this README.md, I have yet to generate a redistributable
* fiserv-vault.yml
  * The manifest used to deploy this release
* jobs
  * Directory that houses all the possible jobs in the deployment. Instances in your deployment can run multiple jobs for this vault deploymet: the consul_server VMs will only run consul_server jobs, vault VMs will run consul_client and vault jobs. Errands are jobs without monit scripts. A smoke-test instance is also recomended and exampled in the deployment manifest. You can create the job skeletons by using `bosh generate-job <job_name>`
* monit
  * This is the monit script for the job. The four lines are as follow, not sure if anything after the first line is ordered
    1. check process <name to display in monit?>
    * with pidfile </path/to/file>
      * file that contains only the int of pid - monit watches the pid in this file to check if the program is running, monit will restart the program if the pid dies without monit's knowledge. If the pid in this file is running monit will think the program is running successfully
    * start program </path/to/ctl_script params>
    * stop program </path/to/ctl_script params>
    * group vcap
* spec (from job)
  * templates
    * all templates that you want evaluated by erb and moved must be listed in here
    * all packages that this job depends on must be listed in here
    * all properties that this jobs depends on must be listed in here
    * I'm somewhat unclear on the consumes and provides but it is how the jobs can use variables and information from each other
* templates
  * I arbitrarily created folders in here just to keep myself sane
  * All files that you want in the /var/vcap/jobs/name directory MUST be in here - I tried just putting the files directly in /var/vcap/jobs/name/bin directory and they never made it to the final deployment
  * Files in this directory are also interpreted by erb so you can use some pretty sweet scripting to get really advanced config files
* packages
  * You can generate package skeletons by running `bosh generate-package <dependency_name>`
  * Packages are generated from either source code or binaries
* packaging
  * Scripts and makefiles needed to make your blobs / source into a binary. In this release, I simply do an `mv` and mark executable
* spec (from package)
  * The files need to either compile the soruce or the binaries
* src
  * the source files you need to make a package - unimplemented for this release

## included errands

### smoke-tests
The smoke-tests errand does not spin up an additional VM, the errand is co-located on every VM that is deployed in this release. The smoke test really just checks the consul status and greps for `failing`. It's not the greatest smoke-test, but it should confirm that everything can work
### vault-init
Spins up a new VM to unseal the vault. Set properties to indicate the key-shares, key-threshold, and VAULT_ADDR. I started to add a way to add a self-signed cert, but then lost focus becasue we now ;) have legit certs in every envirnment. Its possible that this feature might be needed later.
### vault-is-unsealed
Errand is intended to be co-located on the vault VMs, checks the seal status of all the vault instances, if this errand returns 0: you have all unsealed vaults

## Tie it all together and make it work
### Connect the binaries
We don't want these eating up our git repo, so go fetch the binaries and put them somewhere on your system

```bosh add-blob </path/to/consul> consul_binary```

```bosh add-blob </path/to/vault> vault_binary```

### package a dev release
run ```bosh create-release```

do ```bosh -e sandbox upload-release```

then ```bosh -e sandbox -d fiserv-hashicorp-enterprise-vault-consul deploy fiserv-hashicorp-enterprise-vault-consul.yml```

### Create a tarball
```bosh create-release --final --tarball=~/fiserv-hashicorp-enterprise-vault-consul.tgz```

## this guide is over, you probobly want a working service broker
I created a bosh deploy for the service broker too. Check my repo
