# openNetVM Installation

This guide helps you build and install openNetVM.

1. Check System
--

1. Make sure your NIC is supported by Intel DPDK by comparing the following command's ouptput against DPDK's [supported NIC list](http://dpdk.org/doc/nics).

    ```sh
    lspci | awk '/net/ {print $1}' | xargs -i% lspci -ks %
    ```

2.  Check what operating system you have by typing:
    ```sh
    uname -a
    ```
    your Kernel version should be higher than 2.6.33.

3. Install dependencies
    ```sh
    sudo apt-get install build-essential linux-headers-$(uname -r) git
    ```
4. Assure your kernel suppors uio
    ```sh
    locate uio
    ```

2. Setup Repositories
--

1. Download source code
    ```sh
    git clone https://github.com/sdnfv/openNetVM
    ```

2. Initialize DPDK submodule
    ```sh
    git submodule init && git submodule update
    ```

 **From this point forward, this guide assumes that you are working out of the openNetVM source directory.**

3. Set up Environment
--

1. List DPDK supported architectures:
    ```sh
    ls dpdk/config/
    ```

2. Set environment variable RTE_SDK to the path of the DPDK library.  Make sure that you are in the DPDK directory
    ```sh
    echo export RTE_SDK=$(pwd) >> ~/.bashrc
    ```

3. Set environment variable RTE_TARGET to the target architecture of your system.  This is found in step 3.1
    ```sh
    echo export RTE_TARGET=x86_64-native-linuxapp-gcc  >> ~/.bashrc
    ```

4. Set environment variable ONVM_NUM_HUEGPAGES and ONVM_NIC_PCI.

    ONVM_NUM_HUEGPAGES is a variable specifies how many hugepages are reserved by the user, default value of this is 1024, which could be set using: 
    ```sh
    echo export ONVM_NUM_HUEGPAGES=1024 >> ~/.bashrc
    ```

    ONVM_NIC_PCI is a variable that specifies NIC ports to be bound to DPDK.  If ONVM_NIC_PCI is not specified, the default action is to bind all non-active 10G NIC ports to DPDK.
    ```sh
    export ONVM_NIC_PCI=" 07:00.0 07:00.1 "
    ```
5. Source your shell rc file to set the environment variables:
    ```sh
    source ~/.bashrc
    ```

6. Disable ASLR since it makes sharing memory with NFs harder:
   ```sh
    sudo sh -c "echo 0 > /proc/sys/kernel/randomize_va_space"
    ```

4. Configure and compile DPDK
--

1. Run the [install script](../scripts/install.sh) to compile DPDK and configure hugepages.

    The [install script](../scripts/install.sh) will automatically run the [environment setup script](../scripts/setup_environment.sh), which configures your local environment.  This should be run once for every reboot, as it loads the appropraite kernel modules and can bind your NIC ports to the DPDK driver.

5. Run DPDK HelloWorld Application
--

1. Enter DPDK HelloWorld directory and compile the application:

    ```sh
    cd dpdk/examples/helloworld
    make
    ```

2. Run the HelloWorld application

    ```sh
    sudo ./build/helloworld -l 0,1 -n 1
    ```

    If the last line of output is as such, then DPDK works

    ```sh
    hello from core 1
    hello from core 0
    ```

6. Make and test openNetVM
--

1. Compile openNetVM manager and libraries

    ```sh
    cd onvm
    make
    ```

2. Compile example NFs

    ```sh
    cd examples
    make
    ```

3. Run openNetVM manager

    Run openNetVM manager to use 4 cores (1 for displaying statistics, 1 for NIC RX, 1 for NIC TX, and 1 for NF TX) and to use 1 NIC port:

    `./onvm/go.sh 0,1,2,3,4 1`

    You should see information regarding the port that openNetVM is using, and some initial statistics will be displayed.

4. Run speed_tester NF

    To test the system, we will run the speed_tester example NF.  This NF generates a buffer of packets, and sends them to itself to measure the speed of a single NF TX thread.

    In a new shell, run this command to start the speed_tester and giving it one core, and assigning it a Service ID of 1:

    `./examples/speed_tester/go.sh 5 1`

    Once the NF's initialization is completed, you should see the NF display how many packets it is sending to itself.  Go back to the manager to verify that `client 0` is receiving data.  If this is the case, the openNetVM is working correctly.

7. Configuring environment post reboot
--
After a reboot, you can configure your environment again (load kernel modules and bind the NIC) by running the [environment setup script](../scripts/setup_environment.sh).
 
Also, please double check if the environment variables from [step 3](#3-set-up-environment) are initialized.  If they are not, please go to [step 3](#3-set-up-environment)

Troubleshooting
--

1. **Huge Page Configuration**

    You can get information about the hugepage configuration with:

    `grep -i huge /proc/meminfo`

    If there is not enough or no free memory, there are a few reasons why:

     - The manager crashed, but an NF(s) is still running.
         - In this case, either kill them manually by hitting Ctrl+C or run `sudo pkill NF_NAME` for every NF that you have ran.
     - The manager and NFs are not running, but something crashed without freeing hugepages.
         - To fix this, please run `sudo rm -rf /mnt/huge/*` to remove all files that contain hugepage data.
     - The above two cases are not met, something weird is happening:
         - A reboot might fix this problem and free memory

2. **Binding the NIC to the DPDK Driver**

    You can check the current status of NIC port bindings with

    `sudo ./tools/dpdk_nic_bind.py  --status`

    Output similar to below will show what driver each NIC port is bound to.
    
    ```
    Network devices using DPDK-compatible driver
    ============================================
    <none>

    Network devices using kernel driver
    ===================================
    0000:05:00.0 '82576 Gigabit Network Connection' if=eth0 drv=igb unused=igb_uio *Active*
    0000:05:00.1 '82576 Gigabit Network Connection' if=eth1 drv=igb unused=igb_uio
    0000:07:00.0 '82599EB 10-Gigabit SFI/SFP+ Network Connection' if=eth2 drv=ixgbe unused=igb_uio *Active*
    0000:07:00.1 '82599EB 10-Gigabit SFI/SFP+ Network Connection' if=eth3 drv=ixgbe unused=igb_uio
    ```

    In our example above, we see two 10G capable NIC ports that we could use with description `'82599EB 10-Gigabit SFI/SFP+ Network Connection'`.

    One of the two NIC ports, `07:00.0`, is active shown by the `*Active*` at the end of the line.  Since the Linux Kernel is currently using that port, network interface `eth2`, we will not be able to use it with openNetVM.  We must first disable the network interface in the Kernel, and then proceed to bind the NIC port to the DPDK Kernel module, `igb_uio`:

    `sudo ifconfig eth2 down`

    Rerun the status command, `./tools/dpdk_nic_bind.py --status`, to see that it is not active anymore.  Once that is done, proceed to bind the NIC port to the DPDK Kenrel module:

    `sudo ./tools/dpdk_nic_bind.py -b igb_uio 07:00.0`

    Check the status again, `./tools/dpdk_nic_bind.py --status`, and assure the output is similar to our example below: 

    ```
    Network devices using DPDK-compatible driver
    ============================================
    0000:07:00.0 '82599EB 10-Gigabit SFI/SFP+ Network Connection' drv=igb_uio unused=ixgbe
   
    Network devices using kernel driver
    ===================================
    0000:05:00.0 '82576 Gigabit Network Connection' if=eth0 drv=igb unused=igb_uio *Active*
    0000:05:00.1 '82576 Gigabit Network Connection' if=eth1 drv=igb unused=igb_uio
    0000:07:00.1 '82599EB 10-Gigabit SFI/SFP+ Network Connection' if=eth3 drv=ixgbe unused=igb_uio
    ```

