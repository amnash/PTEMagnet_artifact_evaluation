# PTEMagnet Artifact Evaluation Pack

This repository includes an artifact evaluation pack for PTEMagnet (#111 ASPLOS'21 paper). The artifact consists of:
* a patch implementing PTEMagnet in Linux Kernel v4.19 together with scripts for building clean and modified versions of the Linux kernel 
* a disk image for QEMU containing Ubuntu 16.04 LTS with selected SPEC'17, GPOP, and MLPerf ObjDetect benchmarks with their reference datasets; the disk image also contains scripts for running SPEC'17 and GPOP benchmarks colocated with MLPerf ObjDetect 
* a python scripts for launching and measuring the execution time of SPEC'17 and GPOP benchmarks when running colocated with MLPerf ObjDetect within a virtual machine with and without PTEMagnet
* scripts for installing relevant tools and setting the environment

## Installation
This code is designed to run on a x86 server with 20+ cores running Ubuntu 18.04 LTS. 

### Part 1: clone this repo
```bash
sudo apt-get update
sudo apt-get install git
git clone --recurse-submodules https://github.com/amargaritov/PTEMagnet_artifact_evaluation.git
```

### Part 2: install packages and set environment
This step can take up to 2 hours. 
```bash
cd PTEMagnet_artifact_evaluation
./install/install_all.sh <PATH_TO_DIR_WITH_AT_LEAST_150GB_FREE_SPACE>
source source.sh
```
This script  
* installs all relevant tools & libraries
* builds the _clean_ kernel and the _modified_ kernel with PTEMagnet
* downloads the disk image for a VM (if run outside of Cloudlab)
* sets relevant shell and python environment for scripts automating launching and measuring the execution time of benchmarks
* disables THP on the host machine

**Note that we can provide access to a preconfigured Cloudlab profile (and servers) on which this code was tested. Using the Cloudlab profile accelerates installation and simplifies troubleshooting. If you are interested in running the artifact evaluation on Cloudlab, please email Artemiy <artemiy.margaritov@ed.ac.uk>.**

### Part 3: setting ssh keys for passwordless ssh
Evaluation scripts should be able to passwordlessly ssh to 1) a virtual machine where the benchmarks would run and 2) to the host. In our opinion, the simplest way to achieve passwordless ssh is using ssh keys. To upload an ssh key to the virtual machine
* Generate ssh key with 
```bash
ssh-keygen -t rsa
```
* Boot a virtual machine with the provided disk image
```bash
KERNEL=clean; sudo qemu-system-x86_64 KERNEL=clean; sudo qemu-system-x86_64 -kernel $KERNEL_DIR/linux_$KERNEL/arch/x86/boot/bzImage -boot c -m 64G -hda $IMAGE_DIR/rootfs.img -append "root=/dev/sda rw" -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::6666-:22 --enable-kvm -smp 20 -cpu host -nographic
```
* Wait for machine to boot (login prompt should appear)
* Start a new shell on the host, upload the generated ssh key to a virtual machine 
```bash 
ssh-copy-id -p 6666 user@localhost
```
when asked for password, type `user`
* To allow passwordless login from the host to the host itself upload the ssh key to it
```bash
# This does not work on Cloudlab!
ssh-copy-id $USER@localhost
```
If using Cloudlab, you would need to copy your cloudlab ssh key to the host 
```bash
# on machine you ssh to Cloudlab from
scp -P 22 ~/.ssh/<YOUR_CLOUDLAB_KEY> <CLOUDLAB_USERNAME@CLOUDLAB_MACHINE>:~/.ssh/ 
# on Cloudlab machine
chmod 400 ~/.ssh/<YOUR_COPIED_CLOUDLAB_KEY>
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/<YOUR_COPIED_CLOUDLAB_KEY>
```
Test that you can now passwordlessly login to the host
```bash
ssh $USER@localhost
```
* Shutdown the virtual machine
```bash 
ssh -p 6666 user@localhost 'sudo shutdown -h now' 
```
### Part 4: Disable dynamic CPU power management
Dynamic CPU frequency scaling can lead to variability in performance and can introduce substational noise to the measurements. As a result, to reduce the system jitter in the latency measurement experiments it is better to disable frequency scaling. For this, one needs
* change BIOS settings
   * disable `P states`
   * setting "System Profile Settings" in BIOS -> “CPU power management” needs to be set to “ OS DBPM” instead of “Maximum performance”
(the actual list of BIOS settings to change depends on a server model, please contact Artemiy <artemiy.margaritov@ed.ac.uk> if you face problems -- we may provide a preconfigured Cloudlab machine to you)
* update Linux kernel boot paramenters to change the frequency driver from `intel_pstate` to `acpi driver`
  * edit `/etc/default/grub`: add the following boot parameters to `GRUB_CMDLINE_LINUX_DEFAULT`:
  `usbcore.autosuspend=-1 intel_pstate=disable intel_iommu=on iommu=pt nokaslr rhgb quiet tsc=reliable cpuidle.off=1 idle=poll intel_idle.max_cstate=0 processor.max_cstate=0 pcie_aspm=off processor.ignore_ppc=1`
  As a result, you should have a line like this one
  `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash usbcore.autosuspend=-1 intel_pstate=disable intel_iommu=on iommu=pt nokaslr rhgb quiet tsc=reliable cpuidle.off=1 idle=poll intel_idle.max_cstate=0 processor.max_cstate=0 pcie_aspm=off processor.ignore_ppc=1"
  * update grub settings 
  ```bash
  sudo update-grub
  ```
  * reboot to the updated kernel 
  ```
  sudo reboot
  ```
* install cpufreq
```bash 
sudo apt-get install cpufrequtils
```
* check if `intel_pstate` got changed to `acpi driver`:
```bash
cpufreq-info | grep driver
```
should show `acpi`
* set frequency on all cores to 2GHz
```bash 
$REPO_ROOT/install/freq/cpufreq-set-all -f 2000000
```


## Evaluation 
### Getting all the results for Figure 6
You can launch the evaluation for all the benchmarks with `launch_all_exps`. Firstly, create a screen session 
```bash 
screen -L -Logfile ae_all_log -S ae_ptemagnet_all
```
Then start the evaluation of all benchmarks 
```bash
cd evaluation;
. launch_all_exps.sh
```
and make sure the script is running (the virtual machine should start booting, then after about 40 seconds it should move to prefaulting host memory, then launch MLPerf and a benchmark).
If you get an import problem with `distbenchr` inside the screen run 
```bash 
cd distbenchr
python setup.py install
cd ..
```
and try running `launch_all_exps.sh` again.
You can minimize the screen by pressing CTRL+A and D. You can now log out from the host machine as execution will take time. 
You can open a screen to see what is going on with 
```bash
screen -r ae_ptemagnet_all
```
### Vieview the results
You can view the results generated by `launch_all_exps.sh` by running 
```bash
cd $REPO_ROOT/evaluation
./show_results.sh
```

### Running individual benchmarks (if needed)
```bash
cd ./evaluation/; 
mkdir -p results;
./lauch_exp.py --experiment_tag asplos21_ae --kernel [ clean | modified] --app [ bfs | cc | nibble | pr | gcc | mcf | omnetpp | xz ] --num_experiments <int>  --result_dir <FULL_PATH_DIR_TO_STORE_RESULT_FILES_TO> 
```
This script runs the benchmark specified under the `app` parameter in colocation with MLPerf ObjDetect in a virtual machine under the selected kernel type `num_experiments` number of times. The script outputs (prints) the average execution time of the benchmark in such an environment at the end of its execution. 
For example, the following command will print average execution time for mcf run 10 times on a kernel with PTEMagnet (in colocaiton with MLPerf ObjDetect):
```bash
cd ./evaluation/; mkdir -p results; ./launch_exp.py --experiment_tag asplos21_ae --kernel modified --app mcf --num_experiments 10 --result_dir $(pwd)/results
```
To reproduce the results of Figure 6, one needs to run the script for each benchmark two times: with clean and modified kernel and compare the execution times (`launch_all_exps.sh` automates this process). Note that each of the SPEC'17 benchmarks executes for about 12 minutes. The total running time measurement of the average execution time for one benchmark and one kernel type can be approximately calculated 12 * `num_experiments`.



## Miscellaneous

### Building Linux kernel 
Note that both clean and modified Linux kernels were already built by the installation script. You need to run it only if you want to rebuild them. 
```bash
./install/build_kernels.sh <PATH_TO_WHERE_PUT_KERNELS> 
```
Note that both clean and modified Linux kernels were already built by the installation script. You need to run it only if you want to rebuild them. 

### Downloading the VM image
Note that the VM image is already downloaded by the installation script. You don't need to run it.
```bash
./install/download_vm_disk_image.sh <PATH_TO_WHERE_PUT_IMAGE>
```
### Running a virtual machine manually
If you want to boot a virtual machine with the VM image manually you can run the following command:
```bash 
KERNEL=clean; sudo qemu-system-x86_64 -kernel $KERNEL_DIR/linux_$KERNEL/arch/x86/boot/bzImage -boot c -m 64G -hda $IMAGE_DIR/rootfs.img -append "root=/dev/sda rw" -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::6666-:22 --enable-kvm -smp 20 -cpu host -nographic
```
This command boot a machine with 20 cores and 64GB memory. 
After launching the virtual machine you can ssh into it with
```bash 
ssh -p 6666 user@localhost
```
password is `user`
