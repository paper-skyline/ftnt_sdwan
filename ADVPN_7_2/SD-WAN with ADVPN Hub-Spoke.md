# SD-WAN with ADVPN Hub-Spoke Within Single Region

*last updated 26 Feb 2024*

* This template was created to be run against FortiOS 7.2.x
* *With vpn algorithms recommendations from NSA PP-22-0266 Mar 2022 ver 1.0*
* __Seriously re-consider using TWAMP at all in this thing__ ICMP is the generally the best method to use for health checks, particularly with ADVPN connections, since the child tunnel will inherit the health check method from the parent tunnel.
* Some of these config statements have to be entered in a specific order before subsequent command options are available. For example, `set ike-version 2` must be set before defining IKE proposals and DHGRP proposals.
* Use of IKEv2 is best practice, which requires the use of either `peer-id` or the FTNT proprietary `network-id` option under IPSEC phase1 to differentiate between multiple VPN tunnel connections bound to the same interface. *For example, point-to-point VPN tunnels can be configured to use the same interface and should each be mutually exclusive.* If a client does not specify this option, the default behavior on the remote FortiGate is to select the first configured tunnel.

## Useful Debug Commands

`get vpn ipsec tunnel summary` # This command will show you information about VPN tunnels that have completed phase 1 and phase 2 negotiation

`diag vpn ike gateway list name <tunnel-name>` # This command will show you the interface bindings, interface IP addresses, and phase 1 status (Established or Connecting)

`diag vpn ike gateway flush name <tunnel-name>` # This command will teardown and reset the IKE and IPSEC tunnels

`diag sniffer packet any 'host <remote-peer-ip> and (port 500 or 4500)' 4 0 l` # This command will help determine if IKE and IPSEC traffic is being sent to the remote peer

1. `diag debug reset`
2. `diag debug console timestamp enable`
3. `diag vpn ike log-filter dst-addr4 <remote-peer-ip>`
4. `diag debug app ike -1`
5. `diag debug enable`
*Capture and review terminal output*
6. `diag debug disable`

This series of commands can help you determine if there is a mismatch in IKE phase 1 or phase 2 proposals in addition to potential errors in PSK or PKI certificates.

## Hub Configuration

This template is written with the assumption that the hub will be a device that has a "wan1" and "wan2" interface and will use both of them for the SD-WAN underlay. Furthermore, the underlay transport types between the hub and spokes are assumed to be the same, e.g. DIA to DIA or MPLS to MPLS. This is important to note since a DIA underlay can not form an IPSEC VPN to an MPLS circuit.

In this configuration, there is no SD-WAN configuration on the hub. Return traffic from the hub back to the spoke __should__ traverse the same overlay in order to maintain stickiness and avoid asynchronous routing issues or RPF checks. One approach to ensure this takes place correctly is to supplement the hub-spoke approach with BGP route-tags or communities where the spoke is signaling to the hub its preferred overlay transport path for return traffic. The Hub can then use these tags or communities with SD-WAN policies to select the preferred path to the spoke. Other variations of this method exist, with the most recent iteration involving BGP routing over loopback addresses and [embedding SD-WAN SLA information in the ICMP probes sent from a spoke to a hub](https://docs.fortinet.com/document/fortigate/7.2.7/administration-guide/848259).

Because you have to remove all configuration references to interfaces __before__ you configure SD-WAN, you should *strongly* consider configuring SD-WAN before continuing onwards on the Hub.

### IPSEC Configuration

```ruby
config vpn ipsec phase1-interface
    edit "hub-isp1-p1"
        set type dynamic
        set interface "wan1"
        set ike-version 2
        set peertype any # default
        set net-device disable # The hub is not behind a NAT device; default
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set add-route disable # We do not want to add spoke routes to the hub *This didn't take until after saving and coming back in*
        set auto-discovery-sender enable # Hub is going to be sending spokes ADVPN shortcut info. __This command is critical for ADVPN vs just regular hub-spoke approach__
        set dpd on-demand # only shows on full-config
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set ipv4-start-ip 169.254.1.1 # The tunnel IP endpoint range doesn't need to be advertised in dynamic routing since it's a locally connected interface. In a production environment, you'd normally choose routable RFC1918 address space.
        set ipv4-end-ip 169.254.1.250
        set ipv4-netmask 255.255.255.0
        set psksecret <psk>
        [set certificate <signature>] # If using certificates for authentication instead of PSK
        set network-overlay enable
        set network-id 1 # VPN Gateway Network ID; Hub/Spoke must match, can be 0-255
    next
    edit "hub-isp2-p1"
        set type dynamic
        set interface "wan2"
        set ike-version 2
        set peertype any
        set net-device disable # The hub is not behind a NAT device
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set add-route disable # We do not want to add spoke routes to the hub
        set auto-discovery-sender enable # Hub is going to be sending spokes ADVPN shortcut info
        set dpd on-demand # only shows on full-config
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set ipv4-start-ip 169.254.2.1
        set ipv4-end-ip 169.254.2.250
        set ipv4-netmask 255.255.255.0
        set psksecret <psk>
        [set certificate <signature>] # If using certificates for authentication instead of PSK
        set network-overlay enable
        set network-id 2 # VPN Gateway Network ID; Hub/Spoke must match, can be 0-255
    next
end

config vpn ipsec phase2-interface
    edit "hub-isp1-p2"
        set phase1name "hub-isp1-p1"
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
    next
    edit "hub-isp2-p2"
        set phase1name "hub-isp2-p1"
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
    next
end
```

### VPN Interfaces

In this template, I am using link local addresses for the tunnel interfaces since all iBGP SD-WAN members are considered directly connected interfaces. You can use normal RFC1918 address space instead.

```ruby
config system interface
    edit "hub-isp1-p1"
        set ip 169.254.1.253 255.255.255.255
        set remote-ip 169.254.1.254 255.255.255.0
        set interface "wan1"
        set allowaccess ping # may need to enable TWAMP probe-response on this interface for health check
    next
    edit "hub-isp2-p1"
        set ip 169.254.2.253 255.255.255.255
        set remote-ip 169.254.2.254 255.255.255.0
        set interface "wan2"
        set allowaccess ping # may need to enable TWAMP probe-response on this interface for health check
    next
    edit "loopback-hub"
        set vdom root # This has to be entered for command to take
        set type loopback
        set ip 10.10.255.1 255.255.255.255 # This should be in the network range advertised by the Hub via BGP
        set allowaccess ping probe-response # probe-response for TWAMP health check, may need to be bound to virtual p1 interface
    next
end
```

### BGP Routing

*iBGP is used between hub and spokes for SD-WAN within a single region* This method is known as **BGP Routing Per Overlay** which requires the use of BGP addtional-path and ibgp-multipath. This is in comparison to the relatively newer method of **BGP Routing on Loopback** which has different configurations and considerations.

```ruby
config router bgp
    set as 65000
    set router-id <ipv4> # Best practice to set this to loopback address; by default will take the highest loopback address if not defined
    set ibgp-multipath enable
    set additional-path enable
    set additional-path-select 4 # This depends on the max number of tunnels between spoke and hub and _is not_ the same as bgp max-paths (where typically limited to max 6 paths)
    set graceful-restart enable
    config neighbor-group # For greater routing control, you could split the tunnels into two (or more) different neighbor-groups
        edit "spoke-advpn"
            set link-down-failover enable
            set capability-graceful-restart enable
            #set next-hop-self enable # If the hub is also going to peer with eBGP neighbors, this is required.
            set keep-alive-timer 1 # This is the minimum that can be set for BGP. Default is 60sec.
            set holdtime-timer 3 # This is 3x the keep-alive and is the minimum for BGP. Default is 180sec.
            set additional-path both
            set adv-additional-path 4 # This depends on the max number of tunnels between spoke and hub
            set soft-reconfiguration enable
            set remote-as 65000 # This is what makes it iBGP. Aggregate/Summary Addresses are not compatible with ADVPN!
            set route-reflector-client enable # This enabled the RR pushing all spoke routes to all spoke sites
            [set capability-default-originate enable] # This will advertise a default 0.0.0.0/0 route via BGP
        next
    end
    config neighbor-range
        edit 1
            set prefix 169.254.1.0 255.255.255.0 # Comes from "hub-isp1-p1" spoke virtual interfaces
            set neighbor-group "spoke-advpn"
        next
        edit 2
            set prefix 169.254.2.0 255.255.255.0 # Comes from "hub-isp2-p1" spoke virtual interfaces
            set neighbor-group "spoke-advpn"
        next
    end
    config network
        edit 1
            set prefix 10.10.255.0 255.255.255.0 # Whatever data center subnets that you want to advertise; not a summary address or default-route
        next
        edit 2
            set prefix <ipv4 network> # Whatever data center subnets that you want to advertise; not a summary address or default-route
        next
    end
end
```

### FortiGate as TWAMP Server for SD-WAN Health Check

```ruby
config system probe-response
    set mode twamp
    set security-mode authentication # this may need to be adjusted if not doing control and just default test from spoke
    set password "fortinet" # this password needs to match the spoke client
end # prompt that the daemon will restart, must click 'y' to continue

config firewall service custom
    edit "TWAMP"
        set category "Network Services"
        set udp-portrange 8008
        set tcp-portrange 862
    next
end
```

### Firewall Address Objects for SD-WAN Overlay

```ruby
config firewall address
    edit "spoke-tunnels-isp1"
        set subnet 169.254.1.0 255.255.255.0
    next
    edit "spoke-tunnels-isp2"
        set subnet 169.254.2.0 255.255.255.0
    next
    edit "hub-subnets"
        set subnet 10.10.255.0 255.255.255.0 # should match the bgp prefixes advertised to spokes; inclusive of loopback interface above
    next
    edit "r1-s1" # define a subnet for each spoke in the region
        set subnet <ipv4 network> 
    next
    edit "r1-s2" # define a subnet for each spoke in the region
        set subnet <ipv4 network>
    next
end

config firewall addrgrp
    edit "spoke-tunnels"
        set member spoke-tunnels-isp1 spoke-tunnels-isp2
    next
    edit "region-spokes" # should be each spoke subnet within region that could traverse thru hub
        set member r1-s1 r1-s2
    next
end
```

### Firewall Policies for SD-WAN Spokes

*Before* configuring firewall policies where you have to reference interfaces, if you plan to use SD-WAN on the Hub down the road, configure that first before continuing here! In a production network, you would want to tighten these firewall policies down to necessary services and apply appropriate security profiles.

```ruby
config system settings
    set tcp-session-without-syn enable # This makes it so that you can enable on a per-firewall rule basis
end
config firewall policy
    edit 100
        set comment "allow spoke sites to loopback-hub for SD-WAN health check"
        set name "sdwan_spoke_to_loopback"
        set srcint "hub-isp1-p1" "hub-isp2-p1"
        set dstint "loopback-hub"
        set srcaddr "spoke-tunnels"
        set dstaddr "all"
        set schedule "always"
        set service "ALL_ICMP" "TWAMP"
        set action accept
        set logtraffic all # Maybe ok on the hub side since there's no SD-WAN SLA log on this end
    next
    edit 101
        set comment "allow spoke sites to hub subnets"
        set name "sdwan_spoke_to_hub"
        set srcint "hub-isp1-p1" "hub-isp2-p1"
        set dstint <int towards core>
        set srcaddr "region-spokes"
        set dstaddr "hub-subnets"
        set schedule "always"
        set service "ALL" # Also take into account any SD-WAN Services and Health Checks
        set tcp-session-without-syn all # This is necessary on the hubs for cross-FTG sessions in dual-hub or ADVPN; disables stateful inspection on traffic matching this rule.
        set action accept
        set logtraffic all
        # May also want to enable some basic security profiles to inspect traffic coming into Hub from Spokes
    next
    edit 102
        set comment "allow spoke to spoke traffic via hub"
        set name "sdwan_spoke_to_spoke"
        set srcint "hub-isp1-p1" "hub-isp2-p1"
        set dstint "hub-isp1-p1" "hub-isp2-p1"
        set srcaddr "region-spokes"
        set dstaddr "region-spokes"
        set schedule "always"
        set service "ALL" # Spoke should already be controlling what they allow outbound
        set tcp-session-without-syn all # This is necessary for cross-FTG sessions in dual-hub or ADVPN; disables stateful inspection on traffic matching this rule.
        set action accept
        set logtraffic all # May want to disable this as it could get very noisy and spokes may also be logging already
    next
end
```

## Spoke Configuration

### IPSEC Configuration

```ruby
config vpn ipsec phase1-interface
    edit "spoke-isp1-p1"
        set type static # This is the default
        set interface "wan1"
        set ike-version 2
        set peertype any
        set net-device enable # The spoke side may be behind a NAT device
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dpd on-idle
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set auto-discovery-receiver enable # This will allow the spoke to learn ADVPN shortcuts to other spokes sent via the Hub
        set auto-discovery-shortcuts dependent # This is on by default; causes child tunnels (ADVPN shortcuts) to be removed if the parent tunnel drops
        set remote-gw <public ipv4 for hub-isp1-p1>
        set psksecret <pwd> # If using psk this should match what is set on the Hub above
        [set certificate <signature>] # If using certificates for authentication instead of PSK
        [set dpd-retryinterval 5]
        set network-overlay enable
        set network-id 1 # VPN Gateway Network ID; Hub/Spoke should match, can be 0-255
        set add-route disable # Dynamic routes will be received from the Hub via BGP
    next
    edit "spoke-isp2-p1"
        set typic static # This is the default
        set interface "wan2"
        set ike-version 2
        set peertype any
        set net-device enable # The spoke side may be behind a NAT device
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dpd on-idle
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set auto-discovery-receiver enable # This will allow the spoke to learn ADVPN shortcuts to other spokes sent via the Hub
        set auto-discovery-shortcuts dependent # This is on by default; causes child tunnels (ADVPN shortcuts) to be removed if the parent tunnel drops
        set remote-fw <public ipv4 for hub-isp2-p1>
        set psksecret <pwd> # If using psk this should match what is set on the Hub above
        [set certificate <signature>] # If using certificates for authentication instead of PSK
        [set dpd-retryinterval 5]
        set network-overlay enable
        set network-id 2 # VPN Gateway Network ID; Hub/Spoke should match, can be 0-255
        set add-route disable # Dynamic routes will be received from the Hub via BGP
    next
end

config vpn ipsec phase2-interface
    edit "spoke-isp1-p2"
        set phase1name "spoke-isp1-p1"
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set auto-negotiate enable
    next
    edit "spoke-isp2-p2"
        set phase1name "spoke-isp2-p1"
        set proposal aes256-sha384 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommended setting. This has to match spokes and both sides must be capable of supporting it.
        set auto-negotiate enable
    next
end
```

### VPN Interfaces

The tunnel interfaces won't display their assigned IP address in the GUI or when running the `get system interface` command, but will show when running the `diag address list` command and in other debugging commands.

```ruby
config system interface
    edit "spoke-isp1-p1"
        set type tunnel # this is default from phase2 setting
        set allowaccess ping probe-response # ADVPN shortcut paths will open child-health checks bound to these interfaces dynamically
    next
    edit "spoke-isp2-p1"
        set type tunnel # this is default from phase2 setting
        set allowaccess ping probe-response # ADVPN shortcut paths will open child-health checks bound to these interfaces dynamically
    next
    edit "spoke-loopback"
        set vdom "root" # This has to be entered or else it will not take
        set type loopback
        set ip 10.0.1.1 255.255.255.0 # This should be in the network range advertised by the Hub via BGP
        set allowaccess ping  probe-response
    next
end
```

### FortiGate as TWAMP Server for SD-WAN Health Check

```ruby
config system probe-response
    set mode twamp
    set security-mode authentication # this may need to be adjusted if not doing control and just default test from spoke
    set password "fortinet" # this password needs to match the spoke client
end # prompt that the daemon will restart, must click 'y' to continue

config firewall service custom
    edit "TWAMP"
        set category "Network Services"
        set udp-portrange 8008
        set tcp-portrange 862
    next
end
```

### Firewall Address Objects for SD-WAN Overlay

```ruby
config firewall address
    edit "spoke-tunnels-isp1"
        set subnet 169.254.1.0 255.255.255.0
    next
    edit "spoke-tunnels-isp2"
        set subnet 169.254.2.0 255.255.255.0
    next
    edit "hub-subnets"
        set subnet "10.10.255.0 255.255.255.0" # should match the bgp prefixes advertised to spokes
    next
    edit "r1-s1" # define a subnet for each spoke in the region
        set subnet <ipv4 network> 
    next
    edit "r1-s2" # define a subnet for each spoke in the region
        set subnet <ipv4 network>
    next
end

config firewall addrgrp
    edit "spoke-tunnels"
        set member spoke-tunnels-isp1 spoke-tunnels-isp2
    next
    edit "region-spokes" # should be each spoke subnet within region that could traverse thru hub *excluding* your the local spoke subnet
        set member r1-s1 r1-s2
    next
end
```

### SD-WAN Configuration

* *All existing configuration referencing an interface (or member interface in a zone) has to be removed before assigning the member interface*
* TWAMP is a very accurate measure of link health, but requires more configuration overhead than simple ICMP type health checks. Your milage may vary.

```ruby
config system sdwan
    set status enable
    config zone
        edit "underlay-dia"
        next
        edit "overlay-vpn"
        next
    end
    config members
        edit 1
            set interface "wan1"
            set zone "underlay-dia"
            set status enable
            [set cost] # Used if using a cost based Performance SLA
        next
        edit 2
            set interface "wan2"
            set zone "underlay-dia"
            set status enable
            [set cost] # Used if using a cost based Performance SLA
        next
        edit 3
            set interface "spoke-isp1-p1"
            set zone "overlay-vpn"
            set status enable
            [set cost] # This is used with SD-WAN rules and can influence the routing of traffic.
            set priority 10 # This is used with SD-WAN rules and can influence the routing of traffic.
        next
        edit 4
            set interface "spoke-isp2-p1"
            set zone "overlay-vpn"
            set status enable
            [set cost] # This is used with SD-WAN rules and can influence the routing of traffic.
            set priority 10 # This is used with SD-WAN rules and can influence the routing of traffic.
        next
    end
    config health-check
        edit "hub_twamp"
            set server "10.10.255.1"
            set protocol twamp
            set security-mode authentication
            set password "fortinet" # This should matching hub side password
            set members 3 4 # This matches the member from the overlay-vpn zone
            set update-static-route disable
            config sla
                edit 1
                    set latency-threshold 200 # The following should be adjusted to match the quality of your overlay link
                    set jitter-threshold 50
                    set packetloss-threshold 5
                next
            end
        next
    end
    config service
        edit 1
            set mode sla
            set dst "hub-subnets"
            config sla
                edit "hub_twamp"
                    set id 1
                next
            end
            set priority-members 3 4
        next
    end
end
```

### Static Routes

Default route to 0.0.0.0/0 includes both underlay interfaces and overlay interfaces via zones. In order for SD-WAN rules to direct traffic, there has to be an existing route (or routes) in the active routing table for each SD-WAN zone (`get router info routing-table all`)

```ruby
config router static
    edit 1
        set status enable
        set dst 0.0.0.0 0.0.0.0
        set distance 1
        set comment "default static route for internet access"
        set sdwan-zone "underlay-dia" "overlay-vpn"
    next
end
```

### BGP Routing

```ruby
config router bgp
    set as 65000
    set router-id <ipv4> # Best practice to set this; by default will take the highest loopback address if not defined
    set ibgp-multipath enable
    set additional-path enable
    set additional-path-select 4 # This depends on the max number of tunnels between spoke and hub
    set graceful-restart enable
    config neighbor
        edit "169.254.1.253"
            set advertisement-interval 1 # Adjust as desired; goal is to speed up route convergence over defaults
            set capability-graceful-restart enable
            set soft-reconfiguration enable
            set link-down-failover enable
            set remote-as 65000
        next
        edit "169.254.2.253"
            set advertisement-interval 1 # Adjust as desired; goal is to speed up route convergence over defaults
            set capability-graceful-restart enable
            set soft-reconfiguration enable
            set link-down-failover enable
            set remote-as 65000
        next
    end
    config network
        edit 1
            set prefix 10.0.1.0 255.255.255.0 # Whatever spoke subnets that you want to advertise; not a summary address or default-route
        next
        edit 2
            set prefix <ipv4 network> # Whatever spoke subnets that you want to advertise; not a summary address or default-route
        next
    end
end
```

### Firewall Policies for SD-WAN

```ruby
config firewall policy
    edit 100
        set comment "allow hub and spoke sites to loopback-spoke for SD-WAN health check"
        set name "sdwan_advpn_to_loopback"
        set srcint "overlay-vpn"
        set dstint "loopback-spoke"
        set srcaddr "spoke-tunnels"
        set dstaddr "all"
        set schedule "always"
        set service "ALL_ICMP" "TWAMP"
        set action accept
        set logtraffic all # Maybe ok on the hub side since there's no SD-WAN SLA log on this end
    next
    edit 101
        set comment "allow spoke site to hub subnets"
        set name "sdwan_spoke_to_hub"
        set srcint <int towards core>
        set dstint "overlay-vpn"
        set srcaddr "r1-s1"
        set dstaddr "hub-subnets"
        set schedule "always"
        set service <svs to allow>
        set action accept
        set logtraffic all
    next
    edit 102
        set comment "allow hub subnets to spoke site"
        set name "sdwan_hub_to_spoke"
        set srcint "overlay-vpn"
        set dstint <int towards core>
        set srcaddr "hub-subnets"
        set dstaddr "r1-s1"
        set schedule "always"
        set service "ALL"
        set action accept
        set logtraffic all
    next
    edit 103
        set comment "allow spoke to spoke traffic via advpn"
        set name "sdwan_advpn_oubound"
        set srcint <int towards core>
        set dstint "overlay-vpn"
        set srcaddr "r1-s1"
        set dstaddr "region-spokes"
        set schedule "always"
        set service "ALL"
        set action accept
        set logtraffic all
    next
    edit 104
        set comment "allow spoke to spoke traffic via advpn"
        set name "sdwan_advpn_inbound"
        set srcint "overlay-vpn"
        set dstint <int towards core>
        set srcaddr "region-spokes"
        set dstaddr "r1-s1"
        set schedule "always"
        set service "ALL"
        set tcp-session-without-syn all # This is necessary for cross-FTG sessions in dual-hub or ADVPN; disables stateful inspection on traffic matching this rule.
        set action accept
        set logtraffic all
    next
end
```
