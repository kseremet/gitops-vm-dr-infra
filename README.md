# **GitOps VM DR - Ansible Automation README**

This repository contains all the Ansible playbooks, tasks, templates, and EDA rulebooks required to manage the GitOps-based Disaster Recovery (DR) solution for OpenShift Virtualization.  
This automation is responsible for discovering DR-enabled VMs, setting up storage replication, and populating a Git repository (the "manifests repo") with the declarative state of the DR environment.

## **Core Automation Workflows**

This system consists of several key playbooks designed to run at different times.

1. **Reconciliation `(reconcile_all_vms.yml)` - Scheduled Job:**  

   * **Purpose:** Acts as the primary "self-healing" mechanism.  

   * **Action:** Runs periodically (e.g., hourly). It scans the Source Cluster for all VMs with the `dr.demojam.com/enabled=true` label.  

   * For each VM, it runs the core logic (`tasks/process_single_vm.yml`) which:  

     * Checks the VM's `generation` against the version stored in Git to detect changes.  

     * Initializes the Kustomize directory structure (`base/`, `overlays/`) if it doesn't exist.  

     * Sets up storage replication (e.g., enables RBD mirroring) for the VM's disks.  

     * Exports clean, "power-neutral" manifests (`VM`, `PVC`, `PV`) to the `base/<namespace>/` directory.  

     * Updates the `base/<namespace>/kustomization.yaml` file.  

   * **Outcome:** Ensures that any changes, or VMs missed by live events, are eventually consistent with the Git repository.  

2. **Event-Driven Sync (`export_and_clean_vm_resources.yaml`) - EDA Triggered:**  

   * **Purpose:** Provides rapid, real-time updates for new or modified VMs.  

   * **Action:** Triggered by the EDA rulebook on `VM` `ADDED` or `MODIFIED` events (debounced with `once_after`).  

   * It fetches the *latest* VM state from the cluster (to avoid acting on stale event data).  

   * It calls the same core logic (`tasks/process_single_vm.yml`) as the reconciler to sync the changes for that single VM to Git.  

3. **Garbage Collection (`garbage_collector.yml`) - Scheduled / EDA Triggered:**  

   * **Purpose:** Safely removes all resources associated with a VM that is no longer DR-enabled.  

   * **Action:** Can be run periodically (e.g., daily) and is also triggered by `VM` `DELETED` events.  

   * It gets the list of all currently DR-enabled VMs from the Source Cluster.  

   * It scans the Git repository to find all manifests that currently exist.  

   * It identifies "orphan" manifests (files in Git with no corresponding live VM).  

   * For each orphan PV, it runs the storage cleanup logic (e.g., `rbd mirror image disable` for Ceph).  

   * It deletes the orphan manifests (`vm-*.yaml`, `pvc-*.yaml`, `pv-*.yaml`) from Git.  

   * It cleans up any newly empty `base/<namespace>` and `overlays/<cluster>/<namespace>` directories.  

   * It commits all deletions to Git.  

4. **Failover (`failover.yml`) - Manual Operator Job:**  

   * **Purpose:** Prepares the data plane (storage) for a DR failover. This playbook **DOES NOT** modify Git.  

   * **Action:**  

     * An operator runs this playbook, providing a Kubernetes label selector (`-e vm_selector=...`) to target the VMs/application to fail over.  

     * The playbook reads manifests from the Git repository to identify the target VMs and their PVs.  

     * It connects to the **DR Cluster** and runs storage commands (e.g., `rbd mirror image promote --force` for Ceph) to make the disks writable.  

   * **Outcome:** The storage is ready. The playbook informs the operator that they can now proceed with the "Control Plane" step (editing the Kustomization file in Git).

## **Prerequisites**

* **Ansible Collections:** `kubernetes.core`, `ansible.posix`, `ansible.utils`, `community.general`.  
* **Git:** `git` client must be available on the execution nodes.  
* **Kubernetes:** Kubeconfig files/contexts for both Source and DR clusters must be configured.  
* **Credentials:**  
  * An SSH key or PAT for the manifests Git repository (e.g., `GITHUB_PAT`). This should be stored in an AAP Credential.  
  * Kubeconfig credentials for both clusters, stored in AAP Credentials.

## **Setup & Configuration**

1. **Install Collections:** Ensure all required Ansible collections are available in your execution environment.  

2. **Deploy EDA Rulebook:** Configure and deploy the `vm_monitor.yaml` rulebook to watch the Source Cluster. Point its actions to the appropriate AAP Job Templates.  

3. **Configure AAP Job Templates:**  
   * Create Job Templates for `reconcile_all_vms.yml`, `export_and_clean_vm_resources.yaml`, `garbage_collector.yml`, and `failover.yml`.  
   * Inject the Git and Kubernetes credentials into each template.  
   * Pass the required `target_cluster_name` variable to the templates.  
   * Configure schedules for the `reconcile_all_vms.yml` (e.g., `*/15 * * * *`) and `garbage_collector.yml` (e.g., `0 1 * * *`) jobs.  

4. **Initialize Git Repo:** Before enabling the system, run the `reconcile_all_vms.yml` playbook once manually to populate the manifests Git repository with the initial structure (`ApplicationSet`, Kustomize components) and any existing DR-enabled VMs.

## **Ansible Directory Structure**

```
├── ansible/  
│   ├── playbooks/  
│   │   ├── reconcile_all_vms.yml     # (Scheduled) Full sync of all DR VMs  
│   │   ├── export_and_clean_vm_resources.yaml # (EDA) Syncs a single VM on add/modify  
│   │   ├── garbage_collector.yml   # (Scheduled/EDA) Cleans up deleted VMs  
│   │   ├── failover.yml            # (Manual) Prepares storage for failover  
│   │   └── failback.yml            # (Manual) Prepares storage for failback  
│   ├── tasks/  
│   │   ├── process_single_vm.yml   # (Core Logic) Reusable tasks for processing one VM  
│   │   ├── initialize_git_repo.yml # (Core Logic) Creates Kustomize structure  
│   │   ├── git_commit_push.yml     # (Core Logic) Reusable tasks for Git commit  
│   │   ├── setup_ceph_replication.yml # (Storage) Enables RBD mirroring  
│   │   ├── promote_ceph_replication.yml # (Storage) Promotes RBD image  
│   │   ├── cleanup_orphan_file.yml # (Storage) Cleans up a single orphan  
│   │   └── cleanup_empty_dir.yml   # (Storage) Cleans up empty directories  
│   └── templates/  
│       ├── applicationset.yaml.j2  
│       ├── namespace_kustomization.yaml.j2  
│       ├── component_kustomization.yaml.j2  
│       ├── power_on.yaml.j2  
│       └── power_off.yaml.j2  
├── rulebooks/ 
│   ├── vm_monitor.yaml    # EDA rulebook on VM ADDED, MODIFIED and DELETED events  

```
