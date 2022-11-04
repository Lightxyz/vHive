# How to collect and extract vHive logs to local machine


This guide describes how to collect and extract logs in an N-node vHive serverless cluster with Firecracker MicroVMs.

There are a couple of ways to gather and extract logs for vHive to your local machine, whether it is on a single-node cluster or a multi node one. We will present one method to do it. Firstly, if you follow the steps from the [Quickstart guide], logs should already be generated in the `/tmp/vhive-logs` folder at different steps in the workflow. However, sometimes because of running some commands with `screen`, the log folder ends up empty. To mitigate this, you can run the same commands using different `tmux panes`.

> **Note:** `tmux` serves the same role as `screen`, being a terminal multiplexer offering similar features. However, this suggested approach differs a little from the Quickstart's guide approach as we will use different `tmux panes` and not different `tmux sessions`, which are equivalent to terminals.


The following section will describe how to set up a Serverless (Knative) Cluster using `tmux`. This section follows the steps from the Quickstart guide with minor modifications for the log generation and running commands without `screen`. We present how to set up a multi-node cluster, however, the same modifications can be used to generate and extract logs from a single-node cluster.

### 1. Setup All Nodes
**On each node (both master and workers)**, execute the following instructions **as a non-root user with sudo rights** using **bash**:
1. Open a new tmux session
   ```bash
    tmux new -s vhiveSession
    ```
3. Clone the vHive repository
    ```bash
    git clone --depth=1 https://github.com/vhive-serverless/vhive.git
    ```
2. Change your working directory to the root of the repository:
    ```bash
    cd vhive
    ```
3. Create a directory for vHive logs:
    ```bash
    mkdir -p /tmp/vhive-logs
    ```
3. Run the node setup script:
    ```bash
    ./scripts/cloudlab/setup_node.sh > >(tee -a /tmp/vhive-logs/setup_node.stdout) 2> >(tee -a /tmp/vhive-logs/setup_node.stderr >&2)
    ```
    > **BEWARE:**
    >
    > This script can print `Command failed` when creating the devmapper at the end. This can be safely ignored.

### 2. Setup Worker Nodes
**On each worker node**, execute the following instructions **as a non-root user with sudo rights** using **bash**:
1. Run the script that setups kubelet:
    ```bash
    ./scripts/cluster/setup_worker_kubelet.sh > >(tee -a /tmp/vhive-logs/setup_worker_kubelet.stdout) 2> >(tee -a /tmp/vhive-logs/setup_worker_kubelet.stderr >&2)
    ```
2. Start `containerd`:
    ```bash
    sudo containerd 2>&1 | tee /tmp/vhive-logs/containerd_log.txt
    ```
    
    > **Note:**
    >
    > The logs from `containerd` will be generated at this path `/tmp/vhive-logs/containerd_log.txt`. You can change the log's file name or directory. The same applies for all the log files used in the following commands.
    
3. Open a new tmux pane with <kbd>Ctrl</kbd>+<kbd>B</kbd> then <kbd>"</kbd> or <kbd>Ctrl</kbd>+<kbd>B</kbd> then <kbd>%</kbd> then start `firecracker-containerd`:
     > **Note:**
    > You use <kbd>Ctrl</kbd>+<kbd>B</kbd> then <kbd>"</kbd> or <kbd>Ctrl</kbd>+<kbd>B</kbd> then <kbd>%</kbd> to split the current `tmux` pane horizontally or vertically. Additionally you can use <kbd>Ctrl</kbd>+<kbd>B</kbd> then <kbd>→</kbd>, <kbd>←</kbd>, <kbd>↑</kbd> or <kbd>↓</kbd> to navigate between panes.

    ```bash
    sudo PATH=$PATH /usr/local/bin/firecracker-containerd --config /etc/firecracker-containerd/config.toml 2>&1 | tee /tmp/vhive-logs/firecracker-containerd_log.txt
    ```
    

4. Build vHive host orchestrator:
    ```bash
    source /etc/profile && go build
    ```
5. Open a new tmux pane with the same command as the previous step and start `vHive`:
    ```bash
    sudo ./vhive -dbg 2>&1 | tee /tmp/vhive-logs/vhive_log.txt
    ```
    > **Note:**
    >
    > By default, the microVMs are booted and vHive is started in debug mode enabled by `-dbg` flag. Additionally, you can enable snapshots after the 2nd invocation of each function by adding the `-snapshots` flag.
    >
    > If `-snapshots` and `-upf` are specified, the snapshots are accelerated with the Record-and-Prefetch (REAP) technique that we described in our ASPLOS'21 paper ([extended abstract][ext-abstract], [full paper](papers/REAP_ASPLOS21.pdf)).

### 3. Configure Master Node
**On the master node**, execute the following instructions below **as a non-root user with sudo rights** using **bash**:
1. Start `containerd`:
    ```bash
    sudo containerd 2>&1 | tee /tmp/vhive-logs/containerd_log.txt
    ```
2. Split the `tmux` pane and run the script that creates the multinode cluster:
    ```bash
    ./scripts/cluster/create_multinode_cluster.sh > >(tee -a /tmp/vhive-logs/create_multinode_cluster.stdout) 2> >(tee -a /tmp/vhive-logs/create_multinode_cluster.stderr >&2)
    ```
    
    > **BEWARE:**
    >
    > The script will ask you the following:
    > ```
    > All nodes need to be joined in the cluster. Have you joined all nodes? (y/n)
    > ```
    > **Leave this hanging in the terminal as we will go back to this later.**
    >
    > However, in the same terminal you will see a command in the following format:
    > ```
    > kubeadm join 128.110.154.221:6443 --token <token> \
    >     --discovery-token-ca-cert-hash sha256:<hash>
    > ```
    > Please copy both lines of this command.

### 4. Configure Worker Nodes
**On each worker node**, split the `tmux` pane and execute the following instructions **as a non-root user with sudo rights** using **bash**:

1. Add the current worker to the Kubernetes cluster, by executing the command you have copied in step (3.2) **using sudo**:
    ```bash
    sudo kubeadm join IP:PORT --token <token> --discovery-token-ca-cert-hash sha256:<hash> > >(tee -a /tmp/vhive-logs/kubeadm_join.stdout) 2> >(tee -a /tmp/vhive-logs/kubeadm_join.stderr >&2)
    ```
    > **Note:**
    >
    > On success, you should see the following message:
    > ```
    > This node has joined the cluster:
    > * Certificate signing request was sent to apiserver and a response was received.
    > * The Kubelet was informed of the new secure connection details.
    > ```

### 5. Finalise Master Node
**On the master node**, execute the following instructions below **as a non-root user with sudo rights** using **bash**:

1. As all worker nodes have been joined, and answer with `y` to the prompt we have left hanging in the terminal.
2. As the cluster is setting up now, wait until all pods show as `Running` or `Completed`:
    ```bash
    watch kubectl get pods --all-namespaces
    ```

**Your Knative cluster is now ready for deploying and invoking functions.**

Now after you deploy and invoke the functions as instructed in the [Quickstart guide][deploy], you will be able to see all your logs in the `/tmp/vhive-logs` directory. Additionally, you can extract the logs from the nodes to your local machine by running this command on your local machine for each node of the cluster:
 ```bash
 scp -i PATH_TO_SSH_KEY -P 22 -r USERNAME@HOST_NAME:/tmp/vhive-logs PATH_TO_LOCAL_DIRECTORY/SUB_DIR
 ```
   > **Note:**
   >
   > PATH_TO_SSH_KEY represents the path to your ssh key used to connect to the cluster nodes.
   > 
   > USERNAME represents your username for connecting to the cluster nodes.
   >
   > HOST_NAME represents the address of the cluster node you want to extract the logs from.
   >
   > PATH_TO_LOCAL_DIRECTORY represents the local path where you would like to store the logs folder.
   >
   > SUB_DIR represents the sub-directory in which you will store the logs folder. We need such a folder as the logs folder has the same path on every node, in our case `/tmp/vhive-logs`, so in order to not overwrite the local folder we need to use a sub-directory. **Be sure to change the sub-directory name  when running the command for each node to avoid overwriting**.
   >
   > Additionally, you can change the location where the logs are generated as instructed in previous steps, therefore you will need to modify `/tmp/vhive-logs` from the command with the new path where the logs were generated in.
  
  
[Quickstart guide]: https://github.com/vhive-serverless/vHive/blob/main/docs/quickstart_guide.md#ii-setup-a-serverless-knative-cluster
[deploy]: https://github.com/vhive-serverless/vHive/blob/main/docs/quickstart_guide.md#iv-deploying-and-invoking-functions-in-vhive
[ext-abstract]: https://asplos-conference.org/abstracts/asplos21-paper212-extended_abstract.pdf
