# Deploy Active Response Script on Endpoints from Wazuh Manager

## Objective
This guide shows how to deploy a custom Wazuh active response script from a Wazuh manager to Linux and Windows endpoints, and how to configure the manager and agents so the script is pushed and executed remotely.

## Requirements
- Wazuh dashboard administrator access
- SSH or console access to the Wazuh manager
- One-time administrator access to endpoints (Linux or Windows)
- `jq` installed on Linux endpoints
- Python and pip installed on Windows endpoints

## Overview
The workflow includes:
1. Enabling remote command execution on agents
2. Creating an agent group on the manager
3. Adding the active response script to the shared manager path
4. Deploying the script with the `command` module
5. Configuring the active response in `ossec.conf`

## Step 1: Enable remote commands on endpoints

### On Linux
On each Linux endpoint connected to Wazuh, enable remote command execution and restart the Wazuh agent.

Run as root or with `sudo`:

```bash
sudo sh -c 'echo "sca.remote_commands=1" >> /var/ossec/etc/local_internal_options.conf'
sudo sh -c 'echo "wazuh_command.remote_commands=1" >> /var/ossec/etc/local_internal_options.conf'
sudo systemctl restart wazuh-agent
```

### On Windows
Run the below command in PowerShell as Administrator:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\local_internal_options.conf"
```

Then append the below lines into new lines and save the file:

```
sca.remote_commands=1
wazuh_command.remote_commands=1
```

This allows the manager to send commands to the agent.

> ⚠️ Warning:
> Enabling this option allows the Wazuh manager to execute commands on the endpoints. This can introduce security risks if not properly controlled. Ensure that only trusted users have access to the manager, and restrict permissions appropriately to prevent unauthorized command execution on endpoints.

## Step 2: Create an agent group on the manager
1. Open the Wazuh dashboard.
2. Create a Linux or Windows agent group.
3. Assign the agents to that group.

Refer to the [Wazuh documentation](https://documentation.wazuh.com/current/user-manual/agent/agent-management/grouping-agents.html#agent-group-dashboard) if you need more details on creating and managing agent groups.

## Step 3: Add the custom script to the manager
On the Wazuh manager, create or edit the shared script file:

#### For Linux:
```bash
sudo vi /var/ossec/etc/shared/<agent-group>/remove-threat.sh
```

#### For Windows:
```bash
sudo vi /var/ossec/etc/shared/<agent-group>/remove-threat.py
```

Replace `<agent-group>` with your actual group name.

Add the script content needed for your active response and save the file.

For testing, I used the scripts and rules from the [Wazuh malicious file detection and removal documentation](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-remove-malware-virustotal.html#infrastructure).

## Step 4: Configure agent deployment via `agent.conf`
In the Wazuh dashboard, open:

`Agents management` → `Groups` → select your group → `Files` → `agent.conf`

Add this block inside the `<agent_config>` section:

#### For Linux

```xml
<wodle name="command">
  <disabled>no</disabled>
  <tag>deploy-remove-threat</tag>
  <command>/bin/bash -c "cp /var/ossec/etc/shared/remove-threat.sh /var/ossec/active-response/bin/; chmod 750 /var/ossec/active-response/bin/remove-threat.sh; chown root:wazuh /var/ossec/active-response/bin/remove-threat.sh; systemctl restart wazuh-agent"</command>
  <interval>1d</interval>
  <ignore_output>yes</ignore_output>
  <run_on_start>yes</run_on_start>
  <timeout>120</timeout>
</wodle>
```

#### For Windows

```xml
<wodle name="command">
  <disabled>no</disabled>
  <tag>deploy-remove-threat</tag>
  <command>powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "cd 'C:\Program Files (x86)\ossec-agent\shared'; python -m PyInstaller -F remove-threat.py -n remove-threat; Copy-Item '.\dist\remove-threat.exe' 'C:\Program Files (x86)\ossec-agent\active-response\bin\remove-threat.exe' -Force"</command>
  <interval>1d</interval>
  <ignore_output>yes</ignore_output>
  <run_on_start>yes</run_on_start>
  <timeout>600</timeout>
</wodle>
```

Save the agent configuration. The command will copy the script from the shared manager path to the agent active response folder and restart the agent.

> After the script is deployed, change `<disabled>no</disabled>` to `<disabled>yes</disabled>` and save the agent configuration again to prevent repeated execution.

## Step 5: Configure active response on the manager
Edit the Wazuh manager configuration file `/var/ossec/etc/ossec.conf` and add the active response command and rule configuration:

#### For Linux:
```xml
<command>
  <name>remove-threat</name>
  <executable>remove-threat.sh</executable>
</command>

<active-response>
  <disabled>no</disabled>
  <command>remove-threat</command>
  <location>local</location>
  <rules_id>87105</rules_id>
</active-response>
```

#### For Windows:
```xml
<command>
  <name>remove-threat</name>
  <executable>remove-threat.exe</executable>
  <timeout_allowed>no</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>remove-threat</command>
  <location>local</location>
  <rules_id>87105</rules_id>
</active-response>
```

Save the file and restart the manager:

```bash
sudo systemctl restart wazuh-manager
```

## Notes
- Replace `87105` with the actual rule ID that should trigger the active response.
- Ensure `jq` is installed on each Linux endpoint before running the active response script.
- Ensure Python and pip are installed on Windows endpoints.
- Use the correct agent group name when copying the script file.

## Conclusion
Following these steps lets you deploy a custom script from the Wazuh manager to agents and configure active response so the manager can trigger it remotely. This reduces manual deployment work on each agent.
