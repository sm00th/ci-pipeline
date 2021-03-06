<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**

- [CI-Pipeline Architecture and Design](#ci-pipeline-architecture-and-design)
  - [Overview](#overview)
  - [Dependencies and Assumptions](#dependencies-and-assumptions)
  - [Infrastructure and Tools](#infrastructure-and-tools)
  - [CI-Pipeline Complete View](#ci-pipeline-complete-view)
  - [Pipeline Stages](#pipeline-stages)
    - [Trigger](#trigger)
    - [Build Package](#build-package)
    - [Functional Tests on Package](#functional-tests-on-package)
    - [Compose OStree](#compose-ostree)
    - [Integration Tests on OStree](#integration-tests-on-ostree)
    - [e2e Conformance Tests on Openshift Clusters](#e2e-conformance-tests-on-openshift-clusters)
    - [Image Generated From Successful Integration Tests On OStree](#image-generated-from-successful-integration-tests-on-ostree)
    - [Image Smoke Test Validation](#image-smoke-test-validation)
  - [Message Bus](#message-bus)
    - [Message Types](#message-types)
      - [Trigger - org.fedoraproject.prod.git.receive](#trigger---orgfedoraprojectprodgitreceive)
      - [Dist-git message example](#dist-git-message-example)
      - [org.centos.prod.ci.pipeline.package.complete](#orgcentosprodcipipelinepackagecomplete)
      - [org.centos.prod.ci.pipeline.package.ignore](#orgcentosprodcipipelinepackageignore)
      - [org.centos.prod.ci.pipeline.package.queued](#orgcentosprodcipipelinepackagequeued)
      - [org.centos.prod.ci.pipeline.package.running](#orgcentosprodcipipelinepackagerunning)
      - [org.centos.prod.ci.pipeline.package.test.functional.complete](#orgcentosprodcipipelinepackagetestfunctionalcomplete)
      - [org.centos.prod.ci.pipeline.package.test.functional.queued](#orgcentosprodcipipelinepackagetestfunctionalqueued)
      - [org.centos.prod.ci.pipeline.package.test.functional.running](#orgcentosprodcipipelinepackagetestfunctionalrunning)
      - [org.centos.prod.ci.pipeline.compose.complete](#orgcentosprodcipipelinecomposecomplete)
      - [org.centos.prod.ci.pipeline.compose.queued](#orgcentosprodcipipelinecomposequeued)
      - [org.centos.prod.ci.pipeline.compose.running](#orgcentosprodcipipelinecomposerunning)
      - [org.centos.prod.ci.pipeline.compose.test.integration.complete](#orgcentosprodcipipelinecomposetestintegrationcomplete)
      - [org.centos.prod.ci.pipeline.compose.test.integration.queued](#orgcentosprodcipipelinecomposetestintegrationqueued)
      - [org.centos.prod.ci.pipeline.compose.test.integration.running](#orgcentosprodcipipelinecomposetestintegrationrunning)
      - [org.centos.prod.ci.pipeline.image.complete](#orgcentosprodcipipelineimagecomplete)
      - [org.centos.prod.ci.pipeline.image.queued](#orgcentosprodcipipelineimagequeued)
      - [org.centos.prod.ci.pipeline.image.running](#orgcentosprodcipipelineimagerunning)
      - [org.centos.prod.ci.pipeline.image.test.smoke.complete](#orgcentosprodcipipelineimagetestsmokecomplete)
      - [org.centos.prod.ci.pipeline.image.test.smoke.queued](#orgcentosprodcipipelineimagetestsmokequeued)
      - [org.centos.prod.ci.pipeline.image.test.smoke.running](#orgcentosprodcipipelineimagetestsmokerunning)
    - [Reporting Results](#reporting-results)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# CI-Pipeline Architecture and Design

## Overview

The CI-Pipeline is designed to take the Fedora dist-git repos that make up the Fedora Atomic Host and build the RPMs and create the OStree composes and images.  Tests will run at different pipeline stages of the pipeline.  Unit tests will run during the building of the RPMs.  Once RPMs are built successfully functional tests will run against these packages.  Once the compose of the OStree and images is complete integration of the entire host image will be executed.  The final piece is to configure an Origin Openshift cluster using the Atomic Openshift Installer on top of the Fedora Atomic Host images and executing the e2e conformance tests.  Once the OStree and images are validated a message will be published on the fedmsg bus under the CentOS CI topic.

The backbone of communication is fedmsg.  Success and failures of the different pipeline stages will be communicated to reviewers/maintainers in Fedora as well as in the CentOS CI-Pipeline.  The reviewers/maintainers can initially choose to &quot;opt-in&quot; and ultimately be a gating requirement.  When a new OStree and image is available and has passed an acceptable level of testing messages will be published and then Red Hat continuous delivery phase will consume these OStrees and images.

## Dependencies and Assumptions

The CI-Pipeline is not an automation framework or automated tests.  CI in general is the continuous integration of components and validating these work together.  In this case being Fedora Atomic Host. The CI-Pipeline accommodates any automation framework and tests.  Unit tests can easily be defined as make targets and can run as part of the build and need minimal resources.  Functional and integration testing may require more robust test beds to execute scenarios.

Setup and invoking of tests will be done using ansible as outlined in [Invoking Tests Using Ansible](https://fedoraproject.org/wiki/Changes/InvokingTestsAnsible)

Setup and invoking of tests will be done using ansible as outlined in Invoking Tests Using Ansible

We assume that there will be tests available to execute otherwise there is nothing besides building, composing, and configuring an Openshift cluster to validate the OStrees and images.  There is currently tests available for many of the Atomic components as well as the Kubernetes e2e conformance tests.

The CI-Pipeline can be defined using Jenkins Job Builder short-term and Jenkins Pipeline (Jenkinsfile) long-term.  This will be the Jenkins 2.0 Pipeline that is already integrated in Openshift [(Openshift Pipelines Deep Dive)](https://blog.openshift.com/openshift-3-3-pipelines-deep-dive/).

We are dependent on the CentOS CI infrastructure for Openshift, Jenkins resources, test resources (bare metal machines, VMs, containers), datahub, and the fedmsg hub.

## Infrastructure and Tools

The CI-Pipeline will use CentOS CI for Jenkins master and slave resources.  We plan to move these into an Openshift environment and back our Jenkins with persistent volumes.  Initially all jobs will be defined using [Jenkins Job Builder](https://docs.openstack.org/infra/jenkins-job-builder/), but the team plans in the long-term to move these to a [Jenkins Pipeline](https://jenkins.io/doc/book/pipeline/) using the Jenkinsfile declarative format.  We will be using the [Jenkins JMS plugin](https://wiki.jenkins-ci.org/display/JENKINS/JMS+Messaging+Plugin) to listen and publish messages on fedmsg.  There will be other plugins used such as the email ext and groovy, but most will be part of the core of Jenkins 2.0

The pipeline will use [Linch-pin](https://github.com/CentOS-PaaS-SIG/linch-pin) for the provisioning of resources.  Linch-pin allows us to provision bare metal and VMs in many infrastructures such as AWS, Openstack, GCE, Duffy, libvirt,  and Beaker.

CentOS CI will have its own fedmsg topic and a hub to route messages that are published.  We also need a Datahub in CentOS CI for persistent results data.

Currently documentation, jobs, tools, etc. are stored at [CentOS CI-Pipeline](https://github.com/CentOS-PaaS-SIG/ci-pipeline).


## CI-Pipeline Complete View
![ci-pipeline-complete-view](ci-pipeline-complete-view.png)
![ci-pipeline-detail](ci-pipeline-detail.png) 

## Pipeline Stages

### Trigger

Once packages are pushed to Fedora dist-git this will trigger a message.  The pipeline will be triggered via the   [Jenkins JMS plugin](https://wiki.jenkins-ci.org/display/JENKINS/JMS+Messaging+Plugin) for dist-git messages on fedmsg.  This could also be manually triggered by a reviewer/maintainer as well.  Only Fedora Atomic Host packages will be monitored for changes currently.  This may broaden in the long-term.

CI Pipeline messages sent via fedmsg for this stage are captured by the topics org.centos.prod.ci.pipeline.package.[queued,ignored].

### Build Package

Once the pipeline is triggered as part of the build process if unit tests exist they will be executed.

The end result is package will be produced to then be used for further testing.  Success or failure will result with a fedmsg back to the Fedora reviewer/maintainer.

CI Pipeline messages sent via fedmsg for this stage are captured by the topics org.centos.prod.ci.pipeline.package.[running,complete].

### Functional Tests on Package

Functional tests will be executed on the produced package from the previous stage of the pipeline if they exist.  This will help identify issues isolated to the package themselves.  Success or failure will result with a fedmsg back to the Fedora reviewer/maintainer.

CI Pipeline messages sent via fedmsg for this stage are captured by the topics org.centos.prod.ci.pipeline.package.test.functional.[queued,running,complete].

### Compose OStree

If functional tests are successful in the previous stage of the pipeline then an OStree compose is generated.

CI Pipeline messages sent via fedmsg for this stage are captured by the topics org.centos.prod.ci.pipeline.compose.[queued,running,complete].

### Integration Tests on OStree

Integration tests are run on the OStree compose.  Success or failure will result with a fedmsg back to the Fedora reviewer/maintainer.  Also, this can trigger the Red Hat continuous delivery process to run more comprehensive testing if desired.

CI Pipeline messages sent via fedmsg for this stage are captured by the topics org.centos.prod.ci.pipeline.compose.test.integration[queued,running,complete].

### e2e Conformance Tests on Openshift Clusters

If integration tests of the images are successful an openshift cluster will be configured using the Atomic Openshift Installer with the new Fedora Atomic Host image as the base system.  Once the cluster is configured Kubernetes conformance tests will run. Success or failure will result with a fedmsg back to the Fedora reviewer/maintainer.  Also, this can trigger the Red Hat continuous delivery process to run more comprehensive testing if desired.

### Image Generated From Successful Integration Tests On OStree

An image will be initially generated at a certain interval when there has been successful integration test execution on an OStree compose. Success or failure will result with a fedmsg back to the Fedora reviewer/maintainer.  Also, this can trigger the Red Hat continuous delivery process to run more comprehensive testing if desired.

CI Pipeline messages sent via fedmsg for this stage are captured by the topics org.centos.prod.ci.pipeline.image.[queued,running,complete].

### Image Smoke Test Validation

The validation left is to make sure the image can boot and more smoke tests may follow if required.  Success or failure will result with a fedmsg back to the Fedora reviewer/maintainer.  Also, this can trigger the Red Hat continuous delivery process to run more comprehensive testing.

CI Pipeline messages sent via fedmsg for this stage are captured by the topics org.centos.prod.ci.pipeline.image.test.smoke.[queued,running,complete].

## Message Bus

Communication between Fedora, CentOS, and Red Hat infrastructures will be done via fedmsg.  Messages will be received of updates to dist-git repos that we are concerned about for Fedora Atomic host.  Triggering will happen from Fedora dist-git. The CI-Pipeline in CentOS infrastructure will build and functional test packages, compose and integration test ostrees, generate and smoke test (boot) images.  We are dependant on CentOS Infrastructure for allowing us a hub for publishing messages to fedmsg.

### Message Types
Below are the different message types that we listen and publish.  There will be different subtopics so we can keep things organized under the org.centos.prod.ci.pipeline.* umbrella. The fact that ‘org.centos’ is contained in the messages is a side effect of the way fedmsg enforces message naming.

Each change passing through the pipeline is uniquely identified by repo, rev, and namespace. 

#### Trigger - org.fedoraproject.prod.git.receive

Below is an example of the message that we will trigger off of to start our CI pipeline.  We concentrate on the commit part of the message.

````
username=jchaloup
stats={u'files': {u'fix-rootScopeNaming-generate-selfLink-issue-37686.patch': {u'deletions': 8, u'additions': 8, u'lines': 16}, u'build-with-debug-info.patch': {u'deletions': 8, u'additions': 8, u'lines': 16}, u'get-rid-of-the-git-commands-in-mungedocs.patch': {u'deletions': 25, u'additions': 0, u'lines': 25}, u'kubernetes.spec': {u'deletions': 13, u'additions': 11, u'lines': 24}, u'remove-apiserver-add-kube-prefix-for-hyperkube-remov.patch': {u'deletions': 0, u'additions': 169, u'lines': 169}, u'.gitignore': {u'deletions': 1, u'additions': 1, u'lines': 2}, u'sources': {u'deletions': 1, u'additions': 1, u'lines': 2}, u'remove-apiserver-add-kube-prefix-for-hyperkube.patch': {u'deletions': 66, u'additions': 0, u'lines': 66}, u'use_go_build-is-not-fully-propagated-so-make-it-fixe.patch': {u'deletions': 5, u'additions': 5, u'lines': 10}, u'Hyperkube-remove-federation-cmds.patch': {u'deletions': 118, u'additions': 0, u'lines': 118}, u'fix-support-for-ppc64le.patch': {u'deletions': 9, u'additions': 9, u'lines': 18}}, u'total': {u'deletions': 254, u'files': 11, u'additions': 212, u'lines': 466}}
name=Jan Chaloupka
namespace=rpms
rev=b0ef5e0207cea46836a49cd4049908f14015ed8d
agent=jchaloup
summary=Update to upstream v1.6.1
repo=kubernetes
branch=f26
path=/srv/git/repositories/rpms/kubernetes.git
seen=False
message=Update to upstream v1.6.1- related: #1422889
email=jchaloup@redhat.com
````

#### Dist-git message example
````
{
  "source_name": "datanommer",  
  "i": 1, 
  "timestamp": 1493386183.0, 
  "msg_id": "2017-b29fa2b4-0600-4f08-9475-5f82f6684bd4", 
  "topic": "org.fedoraproject.prod.git.receive", 
  "source_version": "0.6.5", 
  "signature": "MbQSb1uwzh6UIFKVm+Uxt+56nW/QRH1nOehifxUrbZfiEDEscRdHtb8dj1Skdv7fcZGHhNlR3PGI\nz/4YqPFJjoAM/k60FsnBIIG1gklJaFBM8MloEYauzo/fUK//W99ojk3UPK0lGTIBijG2knbD9t3T\nUMRuDjt45zmGBXHPlR8=\n", 
  "msg": {
    "commit": {
      "username": "trasher", 
      "stats": {
        "files": {
          "sources": {
            "deletions": 1, 
            "additions": 1, 
            "lines": 2
          }, 
          "php-simplepie.spec": {
            "deletions": 5, 
            "additions": 8, 
            "lines": 13
          }, 
          ".gitignore": {
            "deletions": 0, 
            "additions": 1, 
            "lines": 1
          }
        }, 
        "total": {
          "deletions": 6, 
          "files": 3, 
          "additions": 10, 
          "lines": 16
        }
      }, 
      "name": "Johan Cwiklinski", 
      "rev": "81e09b9c83e8550b54a64c7bdb4e5d7b534df058", 
      "namespace": "rpms", 
      "agent": "trasher", 
      "summary": "Last upstream release", 
      "repo": "php-simplepie", 
      "branch": "f24", 
      "seen": false, 
      "path": "/srv/git/repositories/rpms/php-simplepie.git", 
      "message": "Last upstream release\n", 
      "email": "johan@x-tnd.be"
    }
  }
}
````

#### org.centos.prod.ci.pipeline.package.ignore

````
{ 
  "i": 1,
  "msg_id": "2017-9987b043-b558-4b99-96e0-327548248e65",
  "timestamp": 1501615199,
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.package.ignore",   
  "msg": {
      "CI_NAME": "ci-pipeline-trigger",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "16405",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-trigger/16405/",
      "message-content": "",
      "namespace": "rpms",
      "ref": "fedora/f26/x86_64/atomic-host",
      "repo": "electrum",
      "rev": "8346cbfc821c824c5e5b1a42ac92f8b49ea843a4",
      "status": "<SUCCESS/FAILURE/ABORTED>",
      "test_guidance": "''",
      "topic": "org.centos.stage.ci.pipeline.package.ignore",
      "username": "fedora-atomic"
  }
}
````

#### org.centos.prod.ci.pipeline.package.queued

````
{
    "i": 1,
    "msg_id": "2017-aa2a8e84-9631-40fc-8e4d-6fddd0bba1df",
    "timestamp": 1501616700,
    "crypto": "x509",
    "topic": "org.centos.prod.ci.pipeline.package.running",
    "msg": {
        "CI_NAME": "ci-pipeline-trigger",
        "CI_TYPE": "custom",
        "branch": "f26",
        "build_id": "16426",
        "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-trigger/16426/",
        "message-content": "",
        "namespace": "rpms",
        "ref": "fedora/f26/x86_64/atomic-host",
        "repo": "kernel",
        "rev": "c0af8b92a4c5b13b1ba5b305270eb237b5a18ed1",
        "status": "<SUCCESS/FAILURE/ABORTED>",
        "test_guidance": "''",
        "topic": "org.centos.stage.ci.pipeline.package.queued",
        "username": "fedora-atomic"
    }
}
````

#### org.centos.prod.ci.pipeline.package.running

````
{ 
  "i": 1,
  "msg_id": "2017-6ff7d926-8044-44f7-8efe-41fe114ef665",
  "timestamp": 1501608514, 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.package.running",   
    "msg": {
        "CI_NAME": "ci-pipeline-f26",
        "CI_TYPE": "custom",
        "branch": "f26",
        "build_id": "88",
        "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
        "message-content": "",
        "namespace": "modules",
        "ref": "fedora/f26/x86_64/atomic-host",
        "repo": "perl",
        "rev": "0acffe135ebb30fd922cdf509f0da2a4482b4fb0",
        "status": "<SUCCESS/FAILURE/ABORTED>",
        "test_guidance": "''",
        "topic": "org.centos.prod.ci.pipeline.package.running",
        "username": "fedora-atomic"
    }
}
````

#### org.centos.prod.ci.pipeline.package.complete

````
{ 
  "i": 1,
   "msg_id": "2017-7af7f4e6-d3cf-465a-b039-da6e33efc688",
   "timestamp": 1501610296,
   "crypto": "x509", 
   "topic": "org.centos.prod.ci.pipeline.package.complete",   
   "msg": {
        "CI_NAME": "ci-pipeline-f26",
        "CI_TYPE": "custom",
        "branch": "f26",
        "build_id": "88",
        "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
        "message-content": "",
        "namespace": "modules",
        "package_url": "http://artifacts.ci.centos.org/fedora-atomic/f26/repo/perl_repo/perl-5.24.2-393.fc26.896.0acffe1.x86_64.rpm",
        "ref": "fedora/f26/x86_64/atomic-host",
        "repo": "perl",
        "rev": "0acffe135ebb30fd922cdf509f0da2a4482b4fb0",
        "status": "<SUCCESS/FAILURE/ABORTED>",
        "test_guidance": "''",
        "topic": "org.centos.prod.ci.pipeline.package.complete",
        "username": "fedora-atomic"
    }
}
````

#### org.centos.prod.ci.pipeline.package.test.functional.queued

````
{ 
  "i": 1,
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934",
  "timestamp": 1496382015, 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.test.functional.queued",   
  "msg": { 
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "package_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/rawhide/repo/perl_repo/perl-5.26.0-392.fc27.x86_64.rpm"
      "branch": "rawhide",
      "ref": "fedora/rawhide/x86_64/atomic-host",
      "rev": "0546fa18041a8ca223f4f441dc1868fc81ddce0f", 
      "repo": "perl", 
      "namespace": "rpms", 
      "username": "fedora-atomic", 
      "test_guidance": "<comma-separated-list-of-test-suites-to-run>",
      "message-content": "",
      "topic": "org.centos.prod.ci.pipeline.test.functional.queued",
      "status": "<SUCCESS/FAILURE/ABORTED>"
  }
}
````

#### org.centos.prod.ci.pipeline.package.test.functional.running

````
{ 
  "i": 1,
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934",
  "timestamp": 1496382015, 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.test.functional.running",   
  "msg": { 
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "package_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/rawhide/repo/perl_repo/perl-5.26.0-392.fc27.x86_64.rpm"
      "branch": "rawhide",
      "ref": "fedora/rawhide/x86_64/atomic-host",
      "rev": "0546fa18041a8ca223f4f441dc1868fc81ddce0f", 
      "repo": "perl", 
      "namespace": "rpms", 
      "username": "fedora-atomic", 
      "test_guidance": "<comma-separated-list-of-test-suites-to-run>",
      "message-content": "",
      "topic": "org.centos.prod.ci.pipeline.test.functional.running",
      "status": "<SUCCESS/FAILURE/ABORTED>"
  }
}
````

#### org.centos.prod.ci.pipeline.package.test.functional.complete

````
{ 
  "i": 1,
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934",
  "timestamp": 1496382015, 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.test.functional.complete",   
  "msg": { 
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "package_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/rawhide/repo/perl_repo/perl-5.26.0-392.fc27.x86_64.rpm"
      "branch": "rawhide",
      "ref": "fedora/rawhide/x86_64/atomic-host",
      "rev": "0546fa18041a8ca223f4f441dc1868fc81ddce0f", 
      "repo": "perl", 
      "namespace": "rpms", 
      "username": "fedora-atomic", 
      "test_guidance": "<comma-separated-list-of-test-suites-to-run>",
      "message-content": "",
      "topic": "org.centos.prod.ci.pipeline.test.functional.complete",
      "status": "<SUCCESS/FAILURE/ABORTED>"
  }
}
````

#### org.centos.prod.ci.pipeline.compose.running

````
{ 
  "i": 1,
  "timestamp": 1496382015, 
  "msg_id": "2017-9a3e044f-6886-4cfa-b06f-6a0fa73d04a0",
  "timestamp": 1501610301,
  "topic": "org.centos.prod.ci.pipeline.compose.running",   
  "msg": {
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "compose_rev": "'N/A'",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "namespace": "modules",
      "ref": "fedora/f26/x86_64/atomic-host",
      "repo": "perl",
      "rev": "0acffe135ebb30fd922cdf509f0da2a4482b4fb0",
      "status": "<SUCCESS/FAILURE/ABORTED>",
      "test_guidance": "''",
      "topic": "org.centos.prod.ci.pipeline.compose.running",
      "username": "fedora-atomic"
  }
}
````

#### org.centos.prod.ci.pipeline.compose.complete

````
{ 
  "i": 1,
  "msg_id": "2017-8189dfd5-b873-4a03-9d27-6c6ce2aba65f",
  "timestamp": 1501611084, 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.compose.complete",   
  "msg": {
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "compose_rev": "cfd1c7d2a9cb9875bd6e32b186521f30bf6cbfc7a7d6ab81344736215d995813",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "namespace": "modules",
      "ref": "fedora/f26/x86_64/atomic-host",
      "repo": "perl",
      "rev": "0acffe135ebb30fd922cdf509f0da2a4482b4fb0",
      "status": "<SUCCESS/FAILURE/ABORTED>",
      "test_guidance": "''",
      "topic": "org.centos.prod.ci.pipeline.compose.complete",
      "username": "fedora-atomic"
  }
}
````

#### org.centos.prod.ci.pipeline.compose.test.integration.queued

````
{ 
  "i": 1,
  "msg_id": "2017-060b10d1-af28-43fa-81c7-659384c601d8",
  "timestamp": 1501611503, 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.test.integration.queued",   
  "msg": {
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "compose_rev": "cfd1c7d2a9cb9875bd6e32b186521f30bf6cbfc7a7d6ab81344736215d995813",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "namespace": "modules",
      "ref": "fedora/f26/x86_64/atomic-host",
      "repo": "perl",
      "rev": "0acffe135ebb30fd922cdf509f0da2a4482b4fb0",
      "status": "<SUCCESS/FAILURE/ABORTED>",
      "test_guidance": "''",
      "topic": "org.centos.prod.ci.pipeline.compose.test.integration.queued",
      "username": "fedora-atomic"
  }
}
````

#### org.centos.prod.ci.pipeline.compose.test.integration.running

````
{ 
  "i": 1,
  "msg_id": "2017-d7bc9bd3-fdd2-4a6b-ac8c-c8dac6e774e8",
  "timestamp": 1501611508,
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.test.integration.running",   
  "msg": {
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "compose_rev": "cfd1c7d2a9cb9875bd6e32b186521f30bf6cbfc7a7d6ab81344736215d995813",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "namespace": "modules",
      "ref": "fedora/f26/x86_64/atomic-host",
      "repo": "perl",
      "rev": "0acffe135ebb30fd922cdf509f0da2a4482b4fb0",
      "status": "<SUCCESS/FAILURE/ABORTED>",
      "test_guidance": "''",
      "topic": "org.centos.prod.ci.pipeline.compose.test.integration.running",
      "username": "fedora-atomic"
  }
}
````

#### org.centos.prod.ci.pipeline.compose.test.integration.complete

````
{ 
  "i": 1,
  "timestamp": 1496382015, 
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934", 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.test.integration.complete",   
  "msg": {
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "compose_rev": "cfd1c7d2a9cb9875bd6e32b186521f30bf6cbfc7a7d6ab81344736215d995813",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "namespace": "modules",
      "ref": "fedora/f26/x86_64/atomic-host",
      "repo": "perl",
      "rev": "0acffe135ebb30fd922cdf509f0da2a4482b4fb0",
      "status": "<SUCCESS/FAILURE/ABORTED>",
      "test_guidance": "''",
      "topic": "org.centos.prod.ci.pipeline.compose.test.integration.complete",
      "username": "fedora-atomic"
  }
}
````


#### org.centos.prod.ci.pipeline.image.running

````
{ 
  "i": 1,
  "timestamp": 1496382015, 
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934", 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.image.running",   
  "msg": { 
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "image_url": "N/A",
      "image_name": "N/A",
      "type": "qcow2",
      "compose_rev": "c2dcff5e8d4637a6cb19dd0f3e867b48b3d3b6fa0528dd2d64de23169f9221df",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "ref": "fedora/f26/x86_64/atomic-host",
      "rev": "0546fa18041a8ca223f4f441dc1868fc81ddce0f", 
      "repo": "perl", 
      "namespace": "rpms", 
      "username": "fedora-atomic", 
      "test_guidance": "<comma-separated-list-of-test-suites-to-run>",
      "topic": "org.centos.prod.ci.pipeline.image.complete",
      "status": "<SUCCESS/FAILURE/ABORTED>"
  }
}
````

#### org.centos.prod.ci.pipeline.image.complete

````
{ 
  "i": 1,
  "timestamp": 1496382015, 
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934", 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.image.complete",   
  "msg": { 
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "image_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/images/fedora-atomic-26.85-c2dcff5e8d4637a.qcow2",
      "image_name": "fedora-atomic-26.85-c2dcff5e8d4637a.qcow2",
      "type": "qcow2",
      "compose_rev": "c2dcff5e8d4637a6cb19dd0f3e867b48b3d3b6fa0528dd2d64de23169f9221df",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "ref": "fedora/f26/x86_64/atomic-host",
      "rev": "0546fa18041a8ca223f4f441dc1868fc81ddce0f", 
      "repo": "perl", 
      "namespace": "rpms", 
      "username": "fedora-atomic", 
      "test_guidance": "<comma-separated-list-of-test-suites-to-run>",
      "topic": "org.centos.prod.ci.pipeline.image.complete",
      "status": "<SUCCESS/FAILURE/ABORTED>"
  }
}
````

#### org.centos.prod.ci.pipeline.image.test.smoke.running

````
{ 
  "i": 1,
  "timestamp": 1496382015, 
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934", 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.image.test.smoke.running",   
  "msg": { 
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "image_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/images/fedora-atomic-26.85-c2dcff5e8d4637a.qcow2",
      "image_name": "fedora-atomic-26.85-c2dcff5e8d4637a.qcow2",
      "type": "qcow2",
      "compose_rev": "c2dcff5e8d4637a6cb19dd0f3e867b48b3d3b6fa0528dd2d64de23169f9221df",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "ref": "fedora/f26/x86_64/atomic-host",
      "rev": "0546fa18041a8ca223f4f441dc1868fc81ddce0f", 
      "repo": "perl", 
      "namespace": "rpms", 
      "username": "fedora-atomic", 
      "test_guidance": "<comma-separated-list-of-test-suites-to-run>",
      "topic": "org.centos.prod.ci.pipeline.image.test.smoke.running",
      "status": "<SUCCESS/FAILURE/ABORTED>"
  }
}
````

#### org.centos.prod.ci.pipeline.image.test.smoke.complete

````
{ 
  "i": 1,
  "timestamp": 1496382015, 
  "msg_id": "2017-f2b46ca1-1b07-4539-8d86-f7d3cb1b5934", 
  "crypto": "x509", 
  "topic": "org.centos.prod.ci.pipeline.image.test.smoke.complete",   
  "msg": { 
      "CI_NAME": "ci-pipeline-f26",
      "CI_TYPE": "custom",
      "branch": "f26",
      "build_id": "88",
      "build_url": "https://jenkins-continuous-infra.apps.ci.centos.org/job/ci-pipeline-f26/88/",
      "image_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/images/fedora-atomic-26.85-c2dcff5e8d4637a.qcow2",
      "image_name": "fedora-atomic-26.85-c2dcff5e8d4637a.qcow2",
      "type": "qcow2",
      "compose_rev": "c2dcff5e8d4637a6cb19dd0f3e867b48b3d3b6fa0528dd2d64de23169f9221df",
      "compose_url": "http://artifacts.ci.centos.org/artifacts/fedora-atomic/f26/ostree",
      "message-content": "",
      "ref": "fedora/f26/x86_64/atomic-host",
      "rev": "0546fa18041a8ca223f4f441dc1868fc81ddce0f", 
      "repo": "perl", 
      "namespace": "rpms", 
      "username": "fedora-atomic", 
      "test_guidance": "<comma-separated-list-of-test-suites-to-run>",
      "topic": "org.centos.prod.ci.pipeline.image.test.smoke.complete",
      "status": "<SUCCESS/FAILURE/ABORTED>"
  }
}
````

### Reporting Results

Results will be made available in the CentOS CI Datahub.  The Datahub will monitor CI-Pipeline jobs.  The results stored in the CentOS CI Datahub will be pushed to the Red Hat internal Datahub.  Please refer to documentation about the datahub for detail information.

