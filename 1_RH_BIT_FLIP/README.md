# Artifact Evaluation for "TAROT:A CXL SmartNIC-Based Defense Against Multi-bit Errors by Row Hammers Attacks", ASPLOS 2024.
This repository contains artifacts and workflows for reproducing the experiments presented in the ASPLOS 2024 paper by C. Song et al.

# Contents
1. Reproducing "RH-induced Bit Flips". (related Fig. 4, 5, 6 and 7)
2. Post data processing.
3. Bit Error Rate Measurement of ANVIL and TAROT.

# Hardware pre-requisities
Intel(R) Xeon(R) CPU (Code name Haswell or Broadwell)

We has been successfully tested on:
- Intel(R) Xeon(R) CPU E5-2640 v3 @ 2.60GHz (Haswell)
- Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz (Broadwell)

For reverse engineering the mapping function of physical memory to DRAM addresses, it is highly recommended to install one DIMM of DRAM in the slot.

# Software pre-requisities

g++ >= 8,  cmake (>= 3.14),
python (>= 3.0) for Post data process.

# 1. Reproducing "RH-induced Bit Flips".

1) Allocate N * 1GB hugepages on boot time. (e.g., 4 * 1GB)

   - Update grub file
   ```  
   $ sudo vi /etc/default/grub
   ```
   
   ```
   GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=1G hugepagesz=1G hugepages=4"
   ```
   
   ```
   $ sudo update-grub
   ```

   - Mount hugetlbfs

   ```  
   $ sudo vi /etc/fstab
   ```
   
   ```
   none /mnt/huge hugetlbfs pagesize=1G,size=4G 0 0
   ```

   ```  
   $ sudo reboot
   ```

2) Reverse-engineering Physical Address to DRAM Address Mapping
   We use "DRAMA" to reverse engineering the Physical Address to DRAM Address Mapping.
   
   https://github.com/IAIK/drama.git
   (Peter Pessl et al. "DRAMA: Exploiting DRAM Addressing for Cross-CPU Attacks". In 25th USENIX Security Symposium (USENIX Security 16).)

   We provide the mapping function for Haswell and Broadwell server in the modified code of row hammer attacks.

3) Running RH attack program.

   We modiied "BLACKSMITH" Row-hammer fuzzer for N-sided Row-hammer attack with stripe-pattern.

   https://github.com/comsec-group/blacksmith
   (Patrick Jattke et al. "BLACKSMITH: Scalable Rowhammering in the Frequency Domain".)

   - update RH attack code
     
   ```  
   $ git clone https://github.com/comsec-group/blacksmith.git
   $ cd blacksmith
   $ git apply ../1_RH_BIT_FLIP/RH_ATTACK_PATCH/TAROT_mod.patch
   ```

   - Before you run the RH attack program, update Physical to Dram mapping function and supported_cpus

   ```  
   $ vi ./src/Memory/DRAMAddr.cpp

   Update "struct MemConfiguration"
   We updae the mapping function w/ Haswell and Broadwell cpus. We tested on the "E5-2640", "E5-2680"
   ```

   ```  
   $ vi ./src/Blacksmith.cpp

   update supported_cpus list

     std::vector<std::string> supported_cpus = {
         ...
         "E5-2640",
         "E5-2680",
         " your cpu model in "lscpu" "
     };
   ```
   
   - run RH attack program
     
   ```  
   $ mkdir build
   $ cd build
   $ cmake ..
   $ make -j$(nproc)
   // For 1 rank DIMM
   $ sudo ./blacksmith --dimm-id 1 --runtime-limit 259200000 --ranks 1 -a 150
   // For 2 rank DIMM
   $ sudo ./blacksmith --dimm-id 1 --runtime-limit 259200000 --ranks 2 -a 150
   ```

# 2. Post data processing.

   We provide scripts and example result file for post data processing.
   Refer "./1_RH_BIT_FLIP/POST_PROCESSING/README.md"


# 3. Bit Error Rate Measurement of ANVIL and TAROT.

   After running post analysis "UE_OVER_TIME", update the UE PFNs on the RH attack program.
   
   ```
   void TraditionalHammerer::n_sided_hammer(Memory &memory, int acts, long runtime_limit) {
   ...
   }
   ```

   in ./src/Forges/TraditionalHammerer.cpp)

   After than running the modified RH attack program by employing ANVIL and TAROT.
   You can change the refresh interval of TAROT and monitoring interval of ANVIL at the tarot_mod.h and anvil_mod.h, respectively.
   Note that you have to update UE-vulnerable PFN into the static_ue.h in one of TAROT split module to refresh target addresses.

   

# Trouble shooting.

   - Turn off ECC
     
     If System ECC is enabled, Correctable Errors are not shown in the log. Also, when Uncorrectable Error is generated, system will be crashed.
     
     1) ECC off in the bios menu.
     2) ECC off by modifying MSR register using PCM (please refer ./1_RH_BIT_FLIP/troubleshoot/README.md).
        
   - Refresh rate control
     
     Since latest DRAMs have in-dram mitigation logic, it is difficult to reproduce bit-flip.
     Modifying REFRESH rate of DRAM helps to generate bit-flip errors.
     
     Please refer ./1_RH_BIT_FLIP/troubleshoot/README.md
