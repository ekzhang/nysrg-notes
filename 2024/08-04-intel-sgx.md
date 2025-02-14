- Intel SGX — [[August 4th, 2024]]
    - Provide integrity and confidentiality guarantees to sensitive computation on a computer, when all of the privileged software is compromised. Including the kernel and hypervisor!
    - Q: Does it secure both the code and the data, or just the data?
    - Remote computation service verifies an __attestation key__ against an __endorsement certificate__.
        - The attestation key is created by Intel and baked onto the chip. So basically, the hardware cryptographically signs the code running in its enclave.
    - SGX __enclave__ = "secure container" only containing the private data + code.
    - This paper covers SGX 1, but SGX 2 is apparently similar.
    - SGX PRM = processor-reserved memory. The CPU doesn't allow anyone outside of enclaves to access that memory.
        - EPC = Enclave Page Cache, part of PRM
        - OS (untrusted) assigns EPC pages to different enclaves. CPU ensures that each page belongs to exactly one enclave.
        - Initial enclave state is copied from host memory, so it's not private. Initialization creates a __measurement hash__, and users can verify that they are communicating with an enclave that has a specific measurement hash.
        - How are interrupts and traps handled? AEX = asynchronous enclave exit.
    - Criticism due to security concerns, holes in the program, and Intel's hidden licensing behavior due to "launch control" that forces application developers to get permission from Intel before using the feature.
    - Does the PRM live on a special processor region, or on main memory? We don't know yet. Seems like main memory after reading.
    - Section 2: Thorough crash course in computer processors, operating systems, and virtualization.
        - A motherboard is sockets connected by buses. CPUs connected via QPI and with NUMA DDR DRAM, then on the side there's a PCH ("southbridge") with management engine, flash, UEFI, USB, SATA, etc., as well as NIC hardware.
        - CPU package has major components as well, broken down into parts. Fetch/decode into microcode and with various execution units.
        - Caches implemented in hardware. Coherence protocol and ring interconnect diagrams.
    - Q: How does certificate / attestation keys work? What is the chain of trust?
        - Intel runs a root CA and intermediate CAs. Each processor's attestation key is unique, and the manufacturer shares public endorsement certificates.
    - Q: Can you jmp into the EPC, so you can run code as encrypted data?
    - Q: How does the processor provide access to secure memory or secure randomness? Is the actual PRM storage encrypted?
    - Q: Why is defending against cache timing attacks so difficult? How does this change in a post-Spectre world?
    - Q: If any one person gets the attestation key for one computer, does it compromise all other computers via MITM? We feel like this is too obvious to not be addressed.
        - Yes, this is certainly possible with ion beam microscopy on 14 nm chips (chip attack).
        - Defend against it by reducing the value of a single compromise. Tamper-resistant hardware. It sounds like you can verify the manufacturer's certificate metadata. __This__ is why you need to trust Intel — to only sign chips that are bug-free and hard to physically attack.
    - Q: How do you communicate securely with a running enclave?
        - Seems like the process running the enclave can communicate with it, and from there you can use a TLS channel or something. It's unclear exactly how this communication works, but maybe you send data through channels in the EENTER / ERESUME instructions.
    - Related work
        - IBM 4765 secure coprocessor lives inside a Faraday cage enclosure and has software attestation.
        - ARM TrustZone involves separate address translation units. IP cores.
        - TPM = Auxiliary tamper-resistant chip with no CPU modifications. One enclave measures and attests to the measurement of an OS kernel. Verified Boot vs Trusted Boot features of Intel's Boot Guard in UEFI firmware. Prevents people from just reflashing the firmware.
    - Unlike ordinary crypto, which anyone can simulate and run "strong" algorithms, hardware security requires some additional assumptions like having devices that store certain keys, which are __hard to read__ from outside the chip itself (thanks to advanced EE / lithography).
