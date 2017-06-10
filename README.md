# BOSH Release Flasher

``brflash`` is a tool for fast generation of BOSH compatible development releases which afterwards can be uploaded and deployed via the BOSH CLI. Currently the ``brflash`` supports [v1](https://bosh.io/docs/sysadmin-commands.html) and [v2](https://bosh.io/docs/cli-v2.html) of the CLI. The tool has been tested with [cf-release](https://github.com/cloudfoundry/cf-release) version "264" and [diego-release](https://github.com/cloudfoundry/diego-release) version "1.18.1".

## Use case

The default [cf-release provisioning process](https://github.com/cloudfoundry/bosh-lite#deploy-cloud-foundry) usually takes hours if you haven't initialized all submodules in advance and you haven't already downloaded all necessary blobs. The whole process may take 3-5 hours, depending on the "s3" blobstore provider network bandwidth availability at the moment. The slowest part of the process is the release generation process which may take 2-4 hours, mostly due to the slow download from the "s3" blobstore provider. ``brflash`` helps by significantly speeding up the release generation process. In worst case scenario you have just cloned the "cf-release" repo and you have never used the BOSH CLI, i.e. there are no cached blobs on your machine. In this worst case scenario ``brflash`` is able to generate new "tgz" release archive in  approximately 1 hour, assuming you are working on really low-end laptop. The further deployment process (not covered by "brflash") shouldn't take more than 1 additional hour, which makes a total of 2 hours for the whole provisioning process. This is serious improvement compared to 3-5 hours.

The ``brflash`` tool is BOSH release agnostic and should work fine with all [Cloud Foundry release projects](https://github.com/cloudfoundry?utf8=%E2%9C%93&q=release&type=&language=).

## Prerequisites

* Git version 2.0 or higher in order to utilize the ``--jobs`` argument in the ``git submodule update`` command.
* [RVM](http://rvm.io)
* [BOSH CLI v2](https://bosh.io/docs/cli-v2.html) (optional) if for some reason you want to use the newer CLI (v2) instead of the Ruby based CLI (v1).

In general, if you have created & deployed at least one BOSH release successfully, then you already have pretty much everything you need.

## How to use

Assuming you have cloned the BOSH release project in folder ``~/workspace/bosh-release-folder``, this is the command you need to execute:

```
bash --login brflash ~/workspace/bosh-release-folder
```

This is what happens when you run the tool:

1. Some sanity checks are executed.
2. Ruby 2.3 environment is automatically prepared.
3. If no bosh CLI is detected, the Ruby based CLI (v1) is installed.
4. The latest stable tag of the release is fetched.
5. All release submodules are updated in parallel (hardcoded to 20 submodules in parallel). This is the first place where we save a lot of time. Note that this requires Git version 2.0 or higher.
6. The folder ``.final_builds`` is deleted from the release's repository location. This allows us to skip big portion of the blobs that we don't need since (1) the tool aims at building development releases and (2) currently there is no technical way to download the "final" blobs in parallel. This is the second place where we save a lot of time. Some other folders which contain build artifacts are also deleted.
7. The bosh CLI version is detected. Currently ``brflash`` supports CLIs v1 and v2.
8. The "dev" related blobs are downloaded in parallel (hardcoded to 20 blobs in parallel). Usually the default "s3" blobstore provider is slow and by downloading the blobs in parallel we utilize the network bandwidth more efficiently. This is the third place where we save a lot of time.
9. The actual release is created in the usual way. Some packages are generated from source due to missing corresponding blobs but the process is fast compared to the time we waste with the "s3" blobstore provider.
10. Release is created, corresponding "tgz" archive file is generated and instruction message is displayed.

You can upload the generated release by issuing ``bosh upload release`` or by issuing ``bosh -e <env> upload-release``, depending on the BOSH CLI version that you use. Alternatively, you can upload the "tgz" release archive file. The idea is that once you have the "tgz", you can store it somewhere as backup and use it later when you need it.

After you have uploaded the release, you can proceed with the actual BOSH deployment. You can generate the deployment manifest in the usual way you generate the manifest, either before or after you use ``brflash``. Once you have generated the deployment manifest, you can deploy the generated release in the usual way.
