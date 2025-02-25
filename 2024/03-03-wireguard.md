- WireGuard — [[March 3rd, 2024]]
    - Abel is talking about WireGuard. It's a VPN. :D
    - Background is that it's more a __security__ thing, from the background of the author Jason Donenfeld, or so on. But as a system session, we can assume mathematical & cryptography background, but focuses on the actual Linux kernel engineering and data structures.
    - Ultimately, design decisions behind using radix trees and offering an interface that anyone can use, without thinking about it.
    - Paper is written toward an audience using IPsec / OpenVPN. "These solutions suck."
    - Configuration is annoying, needs you to pick a whole cipher suite (__cipher agility__ was important at the time). WireGuard is an opinionated stance on VPNs, less config.
    - Why use a VPN? You want to connect things securely, and not over the public Internet. Sometimes you can also abuse it to control the intermediate hops through a network. (e.g., using a CDN to get better latency to your __League of Legends__ game server.)
    - PoP — point of presence.
    - OSI layer model is mentioned here, especially layer 2 (link layer) and layer 3 (IP).
    - PQ = "post-quantum" cryptography, could affect the security of protocols like Diffie-Hellman, which rely on the hardness of factorization / discrete logarithms.
    - Section 1
        - From the intro — WireGuard tries to be simple, so it emulates a device `wg0` in the Linux kernel. You configure this with your key pair and the public keys of all peers, and it will "just work" after that point, as any IP packets sent to wg0 will be forwarded to the appropriate virtual network peer. The actual communication happens through UDP on a port though.
        - Why is kernel-space WireGuard more efficient? Well, there's no memory copying between packets in kernel memory and user memory. But maybe you can use kernel-bypass tricks, like DPDK.
    - Section 2
        - "Cryptokey routing" — i.e., your routing table is a list of public keys.
        - Internal IP ranges correspond with public keys (hosts).
        - The practice of making internet endpoints optional for each host enables roaming, in the typical setup of having a "server" with a fixed IP address and a "client" on the other end.
    - Section 3
        - The allowed source virtual IPs for peers in WireGuard are bidirectional. It's the allowed IPs for validating incoming packets, but it's also how we sent outbound packets.
        - You need to have a match in the table for a packet to be sent, including Internet endpoint.
        - Question: What if you have A <-> B via WireGuard peers, but then also B <-> C via WireGuard peers? Can A send a message to C, through B? Yes, this will happen if B has C's ip address in its cryptokey routing table. That's just how Linux works. Linux forwards packets it receives to the appropriate network interface (if iptables, iproute2 are used correctly).
        - It does this through __virtual IP routing__. But it can also do it through the public Internet on B, if the destination address from the packet on A is in a public range. For example, this configuration forwards all Internet traffic through one peer: ![](https://firebasestorage.googleapis.com/v0/b/firescript-577a2.appspot.com/o/imgs%2Fapp%2Fekzhang%2F5yknsmS27i.png?alt=media&token=b3037878-ab2e-4357-a40e-3e99303513b9)
    - Section 4
        - The "Basic Usage" section has the first real, concrete configuration we've seen so far using the `ip` command in Linux.
        - It creates a wg0 interface, where with IP block `10.192.112.3/24`. This means the virtual network has 254 hosts, and the current one is `10.192.112.3`.
        - Possible disadvantages of dropping invalid packets: it can cause a timer stall since no response is sent, so we get no feedback on the host that initiated the connection. This could waste time, in addition to not getting any error message for debugging.
        - Limitations: The initial handshake of WireGuard is quite slow unfortunately. Implementing the Internet over WireGuard would be slower than TCP/IP.
    - Section 5.4 - 5.7
        - Three phases: handshake, cookie (optional, not web browser cookies), and transport.
        - Ephemeral versus static key pairs for encryption. The ephemeral key pair is regenerated every few minutes, giving us forward secrecy.
        - AEAD = authenticated encryption with associated data, i.e., ChaCha20.
        - MAC = message authentication code, i.e., Poly1305. There are two of them for a reason.
    - Section 5.1 - 5.3
        - Protocol has a 1-RTT handshake, which does the ECDH key exchange with Curve25519.
        - Timestamps are used to avoid replay attacks on the initial handshake. You are not allowed to replay the timestamp for a given host. It must be increasing.
        - The timestamp also is used later in a small reorder buffer for transport data.
        - To avoid being broken by quantum computing in the future, there is a bonus option to strengthen the handshake with a pre-shared symmetric secret. You could then append another protocol to this, and the details are left for future work.
        - "Cookies" are a mechanism to avoid DoS. The problem is that ECDH is comparatively slower to the overhead of sending a network packet, like over a 10 Gbit connection.
        - The key idea is to __offload compute__ using the cookie.
        - There are two MACs. The first MAC is always sent. The second MAC is sent after getting a cookie reply, and it allows the requester to try its handshake again. This MAC proves the source IP address, and then the initiator does some expensive subcomputation.
        - There are many problems caused by the cookie. No silence (responding to unauthenticated message), cannot be sent in cleartext, and cookie flood. These are addressed.
    - WireGuard is open-source, only 4000 lines of code, and you can read it!
