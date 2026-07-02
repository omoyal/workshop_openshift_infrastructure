## The rule:
```bash
oc {=call to ocp-api}    VERB{get | create | ....}    OBJECT{pod | deploymeent ....}  FLAG
```

<br><br>

### Bash Compilation
```bash
oc completion bash >>/etc/bash_completion.d/oc_completion
```
<br><br>

### show-console
```bash
oc whoami --show-console
```

<br><br>

### Check Cluster state
```bash
oc get nodes
oc get co
oc get mcp
```

<br><br>

### Other
```bash
oc adm top nodes
oc get project
oc get event
```
