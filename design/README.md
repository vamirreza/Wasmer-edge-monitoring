# Resource Discovery


On the second part of the task, I was assigned to design a system with Cloudprober to have dynamic target discovery and have the ability to take certain actions when the nodes/servers become unhealthy (could be a restart, etc). there will also be additional tests involved to monitor the health of the nodes.
First, let's take a look at the dynamic target discovery part of the system:

#### Dynamic Target Discovery
Reading Cloudprober's Documenation/codebase, we find out that Cloudprober has a  **Resource Discovery Service**.
Quoting from the documenation:

> Cloudprober internally defines and uses a protocol called resource discovery service for targets discovery. It helps provide a consistent interface between the targets subsystem, actual resource discovery mechanisms, and probes subsystem. It also provides a way to move targets discovery into an independent process, which can be used to reduce the upstream API traffic.



Unfortunately, Cloudrpober only has GCP and K8s as it's resource providers for RDS. Reading the task assignment, we presume we already have a way get the current list of (active) edge servers. This could over REST, gRPC, etc.
If we want to keep the system in the Cloudprober echo-system, we can develop a service internally in Cloudprober's codebase that interacts with the said API, implementing the RDS protocol.

Below, I've provided an eagle-eye view of the pipline:

![rds](https://gist.githubusercontent.com/vamirreza/ac0d75bc4a6f48052794826b7d115233/raw/c3c72a546d126029ac54a7df04b69e2dc87cc728/rds.svg)



If that's not an option, we can use the `targets_file` directive in Cloudprober.
We can develop a third-party service that interacts with the API mentioned above and have it *generate* a list of targets and have our Cloudprober instance read that file for it's target discovery.
There won't be any need for a manual reload after generating the file, because according to Cloudprober's documenation on the *targets_file* directive, it realizes a change in the file and automatically reloads the targets.


#### Triggering automatic server restarts

According to the task's requirement, we also need to trigger server restarts when when cloudprober checks keep failing on those servers.
We can use Prometheus and the metrics that Cloudprober provides for us to achieve this.
We also need a configuration management tool to run remote executions on the servers. This could be Ansible, Saltstack, etc.
I'll use Ansible in this context for it's ease of use.

To put things into perspective, I'll list the steps that need to be taken in order to achieve the desired state:

  1) Use the  Prometheus exporter provided to us by Cloudprober to collect real-time metrics from our edge nodes.
  2) Define alert firing rules on Prometheus, for example, fire an alert when an HTTP probe keeps failing.
  3) Set alert conditions and action (e.g. webhook)
  4) Start a web service on an Ansible control server, when receiving POST requests from alertmanager, it'll trigger an ansible-playbook.

  For reference, I've followed [this Stackoverflow discussion](https://stackoverflow.com/questions/28321423/i-have-a-ansible-file-i-just-want-to-run-ansible-file-in-flask-api)

  This is what the Flask API would look like:

  ```python
  from flask import Flask, request
from ansible import context
from ansible.cli import CLI
from ansible.module_utils.common.collections import ImmutableDict
from ansible.executor.playbook_executor import PlaybookExecutor
from ansible.parsing.dataloader import DataLoader
from ansible.inventory.manager import InventoryManager
from ansible.vars.manager import VariableManager
import json
app = Flask(__name__)

@app.route('/action', methods=['POST'])
def send():
    print("Trigger the Ansible Playbook...")
    loader = DataLoader()
context.CLIARGS = ImmutableDict(tags={}, listtags=False,     
        listtasks=False, listhosts=False, syntax=False,   
        connection='ssh', module_path=None, forks=100,  
        remote_user='xxx', private_key_file=None,  
        ssh_common_args=None, ssh_extra_args=None, 
        sftp_extra_args=None, scp_extra_args=None, become=True,
        become_method='sudo', become_user='root', verbosity=True,  
        check=False, start_at_task=None)
inventory = InventoryManager(loader=loader, sources= 
        ('./ansible/hosts',))
variable_manager = VariableManager(loader=loader, 
        inventory=inventory,  
        version_info=CLI.version_info(gitinfo=False))
pbex = PlaybookExecutor(playbooks=
        ['./ansible/reboot_server.yml'], 
        inventory=inventory, variable_manager=variable_manager, 
        loader=loader, passwords={})
results = pbex.run()
  ```
  This Python Flask API will be ran on the server that Alert manager will send it's webhooks to on the `/action` path.

  5) Finally, We'll define an ansible-playbook to reboot the server.

To put things into perspective, here's the pipeline for achieving what we want:

![ansible](https://gist.githubusercontent.com/vamirreza/dbb1433de7b0e82f96bec4ee090731dc/raw/9dec7d070f3a97c7a6931701327f17ad983c74b8/ansible.svg)


#### Additional Tests
For additional tests we can implement some of the following:

  1) Memory Usage: Track memory usage to detect potential memory leaks or excessive usage.
  2) Network Latency: Perform network latency checks between servers to identify network-related issues.
  3) Filesystem Usage: Monitor metrics for each mounted filesystem, including disk space usage, inodes usage, and filesystem read/write statistics. This helps you track the health and utilization of your filesystems.
  4) Processes and Threads: Monitor metrics related to the number of running processes and threads on the system. This can help you monitor resource-intensive processes and ensure the system is not overloaded.
  5) TCP and UDP Connections: Monitor metrics related to active TCP and UDP connections, including established, time-wait, and close-wait connections. This gives you insights into network activity and connection usage.

All of these can be achieved with Node Exporter.
For additional tests, like security auditing, we can use tools like Wazuh, Snort, etc