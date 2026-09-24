By default, Slurm saves the .out and .err files of a job to the present working directory (pwd) from which you executed the sbatch command.
Because an HPC cluster consists of multiple separate compute nodes executing these jobs, a shared network filesystem is absolutely mandatory.

For actual heavy data processing and job scratch spaces (/scratch), true HPC clusters use specialized Parallel File Systems (PFS) instead of NFS.

Examples include Lustre, IBM Spectrum Scale (GPFS), or WekaIO.

How it works: Instead of a single server, data is split ("striped") across multiple storage servers and disks concurrently. When your Slurm job writes a massive file, the compute node writes different chunks of that single file to dozens of hard drives at the exact same time, delivering multi-gigabyte or terabyte-per-second throughput. 

How this maps to Kubernetes (K8s Control Plane)
If you are running containerized Slurm on Kubernetes (using a framework like SUNK or Soperator), the principle remains the same, but the implementation uses 

Kubernetes primitives: 

Persistent Volume Claims (PVCs): The Slurm pod configuration will map a ReadWriteMany (RWX) volume to the container.
The Storage Backend: In Kubernetes, that PVC might be backed by cloud-native distributed storage like CephFS (Rook), a managed cloud file service (like AWS EFS or Google Filestore), or a container-native parallel file system overlay. When the container writes its slurm-%j.out file, it writes into that PVC, ensuring you can read it regardless of which pod or node it ran on. 

The Production Storage Blueprint for a Large University HPC
In a massive environment, you never rely on just one type of storage. You will want to design a multi-tiered system combining Parallel File Systems (PFS), High-Performance NFS, and Local NVMe (Burst Buffers).

1. The Heavy Lifter: Parallel File System (/scratch)
Technology: Lustre, IBM Spectrum Scale (GPFS), or WekaIO.
Use Case: This is where Slurm job outputs, model checkpoints, training datasets, and temporary physics simulation steps live.
Why it matters: Standard NFS will crash if 500 GPU nodes try to write to it simultaneously. A PFS spreads data across dozens of storage targets, unlocking hundreds of gigabytes per second in aggregate bandwidth.
University Policy Tip: Scratch storage is usually un-backed-up and has an aggressive purge policy (e.g., any file untouched for 30 days is automatically deleted) to prevent users from hoarding petabytes of data indefinitely.
2. The User Home Directory (/home and /apps)
Technology: Enterprise NFS (like NetApp or VAST Data) or cloud-native scale-out file systems like CephFS.
Use Case: Storing user source code, .bashrc profiles, Conda/Pip environments, and globally installed software modules (using lmod).
Why it matters: Home directories require excellent metadata performance (handling millions of tiny files) and robust backup systems (snapshots), which parallel file systems are traditionally poor at handling.
3. The Local Burst Buffer (/tmp or /localscratch)
Technology: Local NVMe SSDs physically installed inside each compute node.
Use Case: Node-local staging, deep learning data caching, and rapid data reads that don't need to be shared across the network.

Mapping this to your Kubernetes Control Plane
Because you are using Kubernetes as your control plane, you will bridge these physical enterprise storage systems into your containerized Slurm instances using Container Storage Interface (CSI) drivers.
Rather than letting pods provision storage dynamically (which creates thousands of tiny block devices), you will statically map your massive university storage arrays into Kubernetes:
Deploy the CSI Driver: Install the official CSI driver for your storage choice (e.g., the Lustre CSI driver, IBM Spectrum Scale CSI, or NetApp Trident).
Create Persistent Volumes (PVs): Manually map your pre-existing /scratch and /home export paths into Kubernetes as global PVs with ReadWriteMany (RWX) access modes.
Expose to Slurm Pods: Your Slurm Operator (like Soperator or SUNK) will use standard Persistent Volume Claims (PVCs) to mount these directories directly into the login pods, slurmctld controller, and every worker slurmd pod at identical paths (e.g., /scratch inside the container maps to /scratch on the actual storage cluster).

Critical Advice for University HPC Infrastructure
Beware the "Conda Nightmare": University students love installing massive Python/Conda environments in their /home directories. This creates millions of tiny files that can absolutely cripple an NFS server's metadata engine. Strongly consider providing a global, fast caching area or enforcing containerized environments (Apptainer/Singularity) for student workloads.
Network Backbone: Ensure your storage fabric runs on a dedicated high-speed network. If you are using InfiniBand for your GPU inter-connect (e.g., Nvidia Quantum-2), try to run your storage traffic over a dedicated high-bandwidth Ethernet fabric (like 200GbE/400GbE RoCE) so that heavy storage writes don't bottleneck node-to-node training communication.
To tailor this architectural advice, let me know:
What specific hardware/cloud vendor are you building this on, and do you already have a storage vendor in mind (e.g., DDN, NetApp, VAST)?
What types of workloads dominate your university (e.g., massive multi-node AI training, smaller Genomics pipelines, or traditional MPI physics simulations)?
Which Slurm-on-K8s framework (SUNK, Soperator, Slinky, etc.) are you planning to deploy?
