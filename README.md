## Definition of DRA (Dynamic Resource Allocation)

**Dynamic Resource Allocation (DRA)** is a Kubernetes feature that allows applications to request GPUs by specifying their exact hardware requirements—such as GPU model name and VRAM memory size—instead of just asking for a simple number like "1 GPU".

In traditional Kubernetes setups, GPUs were treated as plain numbers (`nvidia.com/gpu: 1`), with no visibility into a card's actual model or memory size. In contrast, DRA treats every physical GPU as an individual device with its own detailed profile (model name, VRAM capacity, PCIe address, and health status).

To manage devices cleanly, DRA divides the workflow into three steps:
1. **Hardware Inventory (`ResourceSlice`):** Worker nodes publish an exact list of the physical GPUs they have in stock.
2. **Workload Requests (`ResourceClaim`):** Pods specify the exact hardware they need using search filters written in Common Expression Language (CEL).
3. **Container Attachment (CDI):** Once Kubernetes matches a pod to a GPU, it uses the Container Device Interface (CDI) to safely attach that specific card into the container.

---

## How AIRuntime Works with DRA: Step-by-Step Technical Execution

The complete lifecycle of an AIRuntime from API call to active GPU execution follows these 7 distinct steps:

### Step 1: Request Validation, Capacity Hold, and Blueprint Submission (`airuntime-service`)

1. **User Action & API Call:** The user submits a request from the UI to `POST /api/v1beta1/orgs/{org_id}/projects/{project_id}/airuntimes` specifying the runtime name, template ID, GPU hardware config ID, GPU count, region, plan ID, persistent storage size, and optional SSH keys.
2. **Template & Container Image Resolution:** `airuntime-service` queries the database for the selected template (`s.queries.GetAIRuntimeTemplate`) to resolve the container image (`template.Image`), application ports (JupyterLab on port 8888), default environment variables, and any private registry credentials.
3. **Authentication, Quotas, and Plan Verification:** Validates user JWT authentication, verifies project compute quotas, and ensures the billing `planId` matches the requested GPU hardware configuration.
4. **Admission Capacity Hold (`inventory-service`):** Calls `inventoryClient.ReserveResources` to place a temporary 5-minute capacity hold in the database (`resource_reservations` table), preventing concurrent GPU VM or model deployment requests from taking the last GPU before this runtime is provisioned.
5. **Database Record Creation:** Inserts a new row into the PostgreSQL `airuntimes` table with initial status `provisioning` and stores the reservation ID.
6. **Build Blueprint (`AIRuntime` Custom Resource):** Calls `BuildAIRuntimeCR()` to assemble the `AIRuntime` CR (`ai.neevcloud.com/v1alpha1`) containing the target namespace, GPU specifications (model, VRAM, count), resolved container image, Ceph persistent storage configuration, and a randomly generated Jupyter password annotation.
7. **Submit Blueprint to Workload Cluster (`k8sClient.Create`):** Posts the `AIRuntime` CR to the Workload Cluster API server via `k8sClient.Create(ctx, cr)`, handing off execution to the `airuntime-operator`.

---

### Step 2: Operator Creates Storage, GPU Claim, and Pod (`airuntime-operator`)

When the `airuntime-operator` sees the new `AIRuntime` CR in the Kubernetes API server, it does three things:

1. **Creates the Storage Mount (PVC):** If persistent storage is requested, it ensures a `PersistentVolumeClaim` is ready to mount at `/data/workspace` so user files are never lost.
2. **Creates the GPU Request (`ResourceClaim`):** It submits a `ResourceClaim` to Kubernetes asking for the exact GPU needed:
   ```
   device.attributes['gpu.nvidia.com'].productName == 'NVIDIA GeForce RTX 5090' &&
   device.capacity['gpu.nvidia.com'].memory.compareTo(quantity('31Gi')) >= 0
   ```
3. **Creates the Pod Object:** It creates the Pod in Kubernetes with container image (`pytorch-jupyter:v2.4`), storage mount (`/data/workspace`), and attaches the GPU `ResourceClaim`. *(The Pod is created in `Pending` state and does not run yet).*

---

### Step 3: Scheduler Matches the Claim to a GPU Node (`kube-scheduler`)

Once the Pod and its `ResourceClaim` are in Kubernetes, `kube-scheduler` handles the matching:

1. **Reads the Claim:** The scheduler sees the pending Pod asking for a GPU via its `ResourceClaim`.
2. **Checks Node Hardware (`ResourceSlice`):** It checks the hardware catalog (`ResourceSlice`) published by all worker nodes to find a node that has a free card matching the GPU model and VRAM.
3. **Locks the GPU:** It finds an available RTX 5090 (`gpu-0`) on `worker-3`, writes `worker-3, gpu-0` into `ResourceClaim.status.allocation`, and locks the card so no other workload can take it.
4. **Places the Pod on the Node:** It binds the Pod to `worker-3` so the node can start preparing the hardware.

---

### Step 4: Node Hardware Preparation (`kubelet` & DRA Driver DaemonSet)

Every GPU worker node in the cluster runs an **NVIDIA DRA driver DaemonSet pod** (the kubelet plugin) that directly interfaces with the node's PCIe hardware.

1. **Kubelet Receives the Pod:** The `kubelet` agent on `worker-3` sees that the Pod has been assigned to its node and recognizes that it requires a DRA resource (`gpu-claim`).
2. **Local gRPC Call (`NodePrepareResources`):** `kubelet` connects to the local **NVIDIA DRA driver DaemonSet pod** running on that same node via a local UNIX socket and executes a `NodePrepareResources` gRPC call.
3. **Hardware Reset & CDI File Generation:** 
   * The NVIDIA DRA driver daemon on `worker-3` validates physical `gpu-0`, inspects its hardware address, and clears any leftover memory from prior workloads.
   * The driver then writes the **CDI (Container Device Interface) JSON file** directly to the host filesystem at `/var/run/cdi/nvidia.com-gpu.json`:
     ```json
     {
       "cdiVersion": "0.5.0",
       "kind": "nvidia.com/gpu",
       "devices": [
         {
           "name": "gpu-0",
           "containerEdits": {
             "deviceNodes": [
               {"path": "/dev/nvidia0", "hostPath": "/dev/nvidia0"},
               {"path": "/dev/nvidiactl", "hostPath": "/dev/nvidiactl"},
               {"path": "/dev/nvidia-uvm", "hostPath": "/dev/nvidia-uvm"},
               {"path": "/dev/nvidia-uvm-tools", "hostPath": "/dev/nvidia-uvm-tools"}
             ],
             "mounts": [
               {
                 "hostPath": "/usr/lib/x86_64-linux-gnu/libcuda.so.1",
                 "containerPath": "/usr/lib/x86_64-linux-gnu/libcuda.so.1",
                 "options": ["ro", "nosuid", "nodev", "bind"]
               }
             ]
           }
         }
       ]
     }
     ```
   * This JSON file is the concrete instruction sheet that tells `containerd` exactly which `/dev/` files to create and which driver libraries to attach to the container.

---

### Step 5: Container Device Attachment and Process Launch (`containerd` & CDI)

The Pod is now on `worker-3`, and the NVIDIA driver has prepared the CDI instructions file. Next, `containerd` starts the container to run the user's requested workload (e.g., PyTorch + JupyterLab).

By default, a container is isolated and cannot see or access the physical GPU on the host server. To connect the GPU to this container, `containerd` reads the CDI file (`/var/run/cdi/nvidia.com-gpu.json`) and unlocks the **operating system gates**:

1. **Gate 1: Creating the Device Files inside the Pod's `/dev/` Directory:**
   * In Linux, **every piece of hardware is represented as a file** inside the `/dev/` directory (for example, `/dev/sda` for a disk).
   * By default, a Pod's container has its own private `/dev/` folder with **zero GPU files**. If PyTorch looks for a GPU, it finds nothing and throws an error: `No CUDA GPUs are available`.
   * **Only the Requested GPU is Added (Strict Isolation):**
     * If the host node has 8 physical GPUs (`gpu-0` through `gpu-7`) and you requested **1 GPU**, the CDI file contains **ONLY** the data for that single assigned GPU (`gpu-0`).
     * It does **NOT** include any other GPUs on the node. The other 7 cards (`/dev/nvidia1` through `/dev/nvidia7`) do not exist inside your container and are completely invisible.
   * `containerd` reads the CDI file and injects only the requested card's files directly into the Pod's private `/dev/` directory:
     * **`/dev/nvidia0`**: The direct data pipeline to physical GPU 0. PyTorch writes to this file to send AI models, weights, and CUDA calculation instructions.
     * **`/dev/nvidiactl`**: The master control switch used to initialize the card, power it up, and check temperature and clock speeds.
     * **`/dev/nvidia-uvm`**: Unified Virtual Memory file that allows CPU RAM and GPU VRAM to share data directly.
     * **`/dev/nvidia-uvm-tools`**: Profiling interface used by performance monitoring tools (like PyTorch Profiler).

2. **Gate 2: Device Security Filter (Cgroups v2):**
   * Even with the files in `/dev/`, the Linux kernel blocks unauthorized hardware access by default.
   * `containerd` sets a security rule that **allows access ONLY to GPU 0** (`gpu-0`). Access to all other GPUs on the machine is strictly blocked.
   * This ensures your container can only use its assigned card and cannot touch any other user's GPU on the server.

3. **Storage Mount:**
   * If persistent storage was requested, `containerd` mounts the Ceph PVC to `/data/workspace`.

4. **Container Entrypoint Execution:**
   * With device nodes injected, security filters set, and storage mounted, `containerd` executes the container entrypoint process:
     ```bash
     jupyter lab --ip 0.0.0.0 --port 8888
     ```
   * The Pod officially transitions from `Pending` to **`Running`**.

---

### Step 6: Interactive Access Routing

1. The container entrypoint executes JupyterLab on internal port `8888`.
2. `airuntime-operator` creates an HTTP route via the Gateway API exposing `https://.../jupyter` secured by `JUPYTER_PASSWORD`.
3. If SSH access was requested, the operator configures SSHPiper to proxy SSH connections to the pod's IP using tenant SSH public keys.
4. The user connects via their browser and executes CUDA code on the bare-metal GPU.

---

### Step 7: Release on Stop (`reconcileStop`)

1. When the user stops the AIRuntime, `airuntime-controller` triggers `reconcileStop`:
   * Deletes the Deployment / Pod.
   * Deletes the `ResourceClaim`.
   * Deleting the claim releases the device lock on `gpu-0`, making it immediately free for other users in the cluster.
   * The PVC and `AIRuntime` custom resource are preserved, ensuring all files, notebooks, and checkpoints are safely kept.


