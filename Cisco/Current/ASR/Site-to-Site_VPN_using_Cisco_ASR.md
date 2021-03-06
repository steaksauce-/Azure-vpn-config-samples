This configuration template applies to **Cisco ASR 1000 Series** Aggregation Services Routers running **IOS XE 15.2** or greater. It configures an IPSec VPN tunnel connecting your on-premise VPN device with the Azure gateway. Things that begin with "azure-" are variable names and can be changed consistently.

- Vpn Type: **RouteBased**
- Local virtual network gateway Ip Address: **131.X.X.X** (Outside Interface IP Address of ASR or Public IP address)
- Local Network Prefix: **10.0.0.0/8** (Your on-premises local network. Specify starting IP address of your network.)
- Azure Virtual Network **192.168.1.0/16**
- Shared Key: **a1af0cdcb0494757a16abe0cc2101c7b**


### ACL rules###

Proper ACL rules are needed for permitting cross-premise network traffic. You should also allow inbound UDP/ESP traffic for the interface which will be used for the IPSec tunnel. In this example **10.0.0.0/8** is the on premises network & **192.168.1.0/16** is the Azure Virtual Network In this example the Azure Gateway IP Address is **40.76.X.X** and your Outside Interface IP Address is **131.X.X.X**

	access-list 101 permit ip 10.0.0.0 0.255.255.255 192.168.0.0 0.0.255.255
	access-list 102 permit udp host 40.76.X.X eq isakmp host 131.X.X.X
	access-list 102 permit esp host 40.76.X.X host 131.X.X.X

### Internet Key Exchange (IKE) configuration ###

This section specifies the authentication, encryption, hashing, and Diffie-Hellman group parameters for the Phase 1 negotiation and the main mode security association. In this example the Azure Gateway IP Address is **40.76.X.X**
	
	crypto ikev2 proposal azure-proposal
	  encryption aes-cbc-256 aes-cbc-128 3des
	  integrity sha1
	  group 2
	  exit
	
	crypto ikev2 policy azure-policy
	  proposal azure-proposal
	  dpd 10 30
	  exit
	
	crypto ikev2 keyring azure-keyring
	  peer 40.76.X.X
	    address 40.76.X.X
	    pre-shared-key a1af0cdcb0494757a16abe0cc2101c7b
		exit
	  exit
	
	crypto ikev2 profile azure-profile
	  match address local interface <NameOfYourOutsideInterface>
	  match identity remote address 40.76.X.X 255.255.255.255
	  authentication remote pre-share
	  authentication local pre-share
	  keyring local azure-keyring
	  exit

### IPSec configuration ###

This section specifies encryption, authentication, tunnel mode properties for the Phase 2 negotiation

	crypto ipsec transform-set azure-ipsec-proposal-set esp-aes 256 esp-sha-hmac
	 mode tunnel
	 exit

### Crypto map configuration ###

This section defines a crypto profile that binds the cross-premise network traffic to the IPSec transform set and remote peer. We also bind the IPSec policy to the virtual tunnel interface, through which cross-premise traffic will be transmitted. We have picked an arbitrary tunnel id **"1"** as an example. If that happens to conflict with an existing virtual tunnel interface, you may choose to use a different id. The IP address **169.254.0.**1 acts as the “inner” address of the tunnel. Essentially it has one job, to deliver traffic from the Azure side to the on-prem side. As it does not need to reach the Internet, it being routable is not necessary. The ASR has an internal routing table and knows what to do with the traffic. You should be able to use any **169.254.X.X** address. 

	crypto ipsec profile azure-vti
	  set transform-set azure-ipsec-proposal-set
	  set ikev2-profile azure-profile
	  exit
	
	int tunnel 1
	  ip address 169.254.0.1 255.255.255.0
	  ip tcp adjust-mss 1350
	  tunnel source <NameOfYourOutsideInterface>
	  tunnel mode ipsec ipv4
	  tunnel destination 40.76.X.X
	  tunnel protection ipsec profile azure-vti
	  exit
	
	ip route 192.168.0.0 255.255.0.0 tunnel 1