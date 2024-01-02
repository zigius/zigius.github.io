---
title: "Emr Studio Notebook Setup"
date: 2024-01-02T19:17:58+02:00
draft: false
---

Common issue we faced when setting up EMR Studio was that we could not connect to the notebook. The error message was:

```
Error: Unable to attach to cluster j-XXXXXXXXXXX. Reason: Attaching the workspace(notebook) failed. Internal error
```

The solution was to run the following command in the EMR cluster after SSH to the master node:

```bash
sudo systemctl start hadoop-httpfs
sudo systemctl status hadoop-httpfs
```

Alternatively, you can add the following EMR step to the cluster:
```
==========
JAR location: command-runner.jar
Main class: None
Arguments: bash -c "sudo systemctl start hadoop-httpfs"
Action on failure: Continue
==========
```

Taken from [here](https://repost.aws/knowledge-center/emr-notebook-connection-issues)
