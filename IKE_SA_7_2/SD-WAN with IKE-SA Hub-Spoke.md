# SD-WAN with IKE-SA Hub-Spoke Within Single Region

* This template was created to be run against FortiOS 7.2.x
* *With vpn algoritihms recommendations from NSA PP-22-0266 Mar 2022 ver 1.0*
* This is a cleaner, and more much scalable, method of establishing SD-WAN shortcuts between Spokes while continuing the best practice of using BGP community tags to signal the preferred return tunnel path from Spoke to Hub. The same 
* Instead of relying on BGP Route Reflection from the Hubs to Spokes, along with sharing a full BGP route table to every Spoke, this method relies on Spokes using IKE Security Association definitions within the IKE Phase II process to advertise local routes. The Hub then brokers shortcuts between Spokes that use static routes between each other instead of BGP routes.

## Hub Configuration

### Firewall Address Objects and Groups for IPSEC and SD-WAN

```ruby
config firewall address
    edit "spoke-tunnels-isp1"
        set subnet 169.254.1.0 255.255.255.0
    next
    edit "spoke-tunnels-isp2"
        set subnet 169.254.2.0 255.255.255.0
    next
    edit "r1-h1-vlan1" # define a subnet for each vlan within the hub
        set subnet <ipv4 network>
    next
    edit "r1-h1-vlan2" # define a subnet for each vlan within the hub
        set subnet <ipv4 network> 
    next
    edit "r1-s1" # define a subnet for each spoke in the region
        set subnet <ipv4 network> 
    next
    edit "r1-s2" # define a subnet for each spoke in the region
        set subnet <ipv4 network>
    next
end

config firewall addrgroup
    edit "spoke-tunnels"
        set member spoke-tunnels-isp1 spoke-tunnels-isp2
    next
    edit "r1-h1-lan"
        set member "r1-h1-vlan1" "r1-h1-vlan2"
    next
    edit "region-spokes" # should be each spoke subnet within region that could traverse thru hub
        set member r1-s1 r1-s2
    next
end
```

### IPSEC Configuration

```ruby
config vpn ipsec phase1-interface
    edit "hub-isp1-p1"
        set type dynamic
        set interface "wan1"
        set ike-version 2
        set peertype any
        set net-device disable # The hub is not beind a NAT device
        set network-id 1
        set proposal aes256-sha384 # NSA recommnended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set add-route disable # We do not want to add spoke routes to the hub
        set auto-discovery-sender enable # Hub is going to be sending spokes ADVPN shortcut info
        set dpd on-demand # only shows on full-config
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set ipv4-start-ip 169.254.1.1 # The tunnel IP endpoint range doesn't need to be advertised in dynamic routing since it's a locally connected interface
        set ipv4-end-ip 169.254.1.250
        set ipv4-netmask 255.255.255.0
        set psksecret <psk>
        [set certificate <signature>] # If using certificates for authentication instead of PSK
        set network-overlay enable
        [set dpd-retrycount 2] # Shortening the DPD health check when tunnel is idle
        [set dpd-retryinterval 2]
    next
    edit "hub-isp2-p1"
        set type dynamic
        set interface "wan2"
        set ike-version 2
        set peertype any
        set net-device disable # The hub is not beind a NAT device
        set network-id 2
        set proposal aes256-sha384 # NSA recommnended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
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
        [set dpd-retrycount 2] # Shortening the DPD health check when tunnel is idle
        [set dpd-retryinterval 2]
    next
end

config vpn ipsec phase2-interface
    edit "hub-isp1-p2"
        set phase1name "hub-isp1-p1"
        set proposal aes256-sha384 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set src-addr-type name
        set dst-addr-type name
        set src-name "r1-h1-lan"
        set dst-name "all"
    next
    edit "hub-isp2-p2"
        set phase1name "hub-isp2-p1"
        set proposal aes256-sha384 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set src-addr-type name
        set dst-addr-type name
        set src-name "r1-h1-lan"
        set dst-name "all"
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
        set allowaccess ping
    next
    edit "hub-isp2-p1"
        set ip 169.254.2.253 255.255.255.255
        set remote-ip 169.254.2.254 255.255.255.0
        set interface "wan2"
        set allowaccess ping
    next
    edit "loopback-hub"
        set vdom root # This has to be entered for command to take
        set type loopback
        set ip 10.10.255.1 255.255.255.255 # This should be in the network range advertised by the Hub via BGP
        set allowaccess ping probe-response # probe-response for TWAMP health check; can be excluded if not using TWAMP
    next
end
```

### SD-WAN Configuration

```ruby
config system sdwan
    set status enable
    config zone
        edit "underlay-dia"
        next
        edit "overlay-vpn"
        next
    end
    config members # Any config reference to these interfaces has to be removed before you can use them!
        edit 1
            set interface "hub-isp1-p1"
            set zone "overlay-vpn"
            set status enable
            [set cost] # Used if using a cost based Performance SLA
        next
        edit 2
            set interface "hub-isp2-p1"
            set zone "overlay-vpn"
            set status enable
            [set cost] # Used if using a cost based Performance SLA
        next
    end
end
```

### BGP Routing

*iBGP is used between hub and spokes for SD-WAN*

```ruby
config router bgp
    set as 65000
    set router-id <ipv4> # Best practice to set this; by default will take the highest loopback address if not defined
    set graceful-restart enable
    set ibgp-multipath enable
    config neighbor-group # For greater routing control, you could split the tunnels into two (or more) different neighbor-groups
        edit "spoke-ike-sa"
            set link-down-failover enable
            set capability-graceful-restart enable 
            set soft-reconfiguration enable
            set remote-as 65000 # This is what makes it iBGP. Aggregate/Summary Addresses are not compatible with ADVPN!  
            [set next-hop-self enable] # Not sure that this is really required for iBGP between spokes
            [set capability-default-originate enable] # This will advertise a default 0.0.0.0/0 route via BGP
        next
    end
    config neighbor-range
        edit 1
            set prefix 169.254.1.0 255.255.255.0 # Comes from "hub-isp1-p1" spoke virtual interfaces
            set neighbor-group "spoke-ike-sa"
        next
        edit 2
            set prefix 169.254.2.0 255.255.255.0 # Comes from "hub-isp2-p1" spoke virtual interfaces
            set neighbor-group "spoke-ike-sa"
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

### Firewall Policies for SD-WAN Spokes

In this configuration template, SD-WAN zones with member interfaces have been created and the following firewall policies reference those. In a production network, you would want to tighten these firewall policies down to necessary services and apply appropriate security profiles.

```ruby
config system settings
    set tcp-session-without-syn enable # This makes it so that you can enable on a per-firewall rule basis
end
config firewall policy
    edit 100
        set comment "allow spoke sites to loopback-hub for SD-WAN health check"
        set name "sdwan_spoke_to_loopback"
        set srcint "overlay-vpn"
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
        set srcint "overlay-vpn"
        set dstint <int towards core>
        set srcaddr "region-spokes"
        set dstaddr "r1-h1-lan"
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
        set srcint "overlay-vpn"
        set dstint "overlay-vpn"
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

### Firewall Address Objects and Groups for IPSEC and SD-WAN

```ruby
config firewall address
    edit "r1-s1-vlan1"
        set subnet <ipv4 network>
    next
    edit "r1-s1-vlan2"
        set subnet <ipv4 network> 
    next
    edit "hub-subnets"
        set subnet "10.10.255.0 255.255.255.0" # should match the bgp prefixes advertised to spokes
    next
end

config firewall addrgroup
    edit "r1-s1-lan"
        set member "r1-s1-vlan1" "r1-s1-vlan2"
    next
end
```

### IPSEC Configuration

```ruby
config vpn ipsec phase1-interface
    edit "spoke-isp1-p1"
        set type static
        set interface "wan1"
        set ike-version 2
        set peertype any
        set net-device enable # The spoke side may be behind a NAT device
        set network-id 1
        set localid "region1-spoke1"
        set proposal aes256-sha384 # NSA recommnended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set add-route enable # This is the default setting so will only show in full. We want to receive routes as part of IKE process.
        set dpd on-idle
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set auto-discovery-receiver enable # This will allow the spoke to learn ADVPN shortcuts to other spokes sent via the Hub
        [set auto-discovery-shortcuts dependent] # This controls whether the Spoke shortcuts should be deleted if the Hub tunnel goes down. If you are also using `tcp-session-without-syn enable` then the failover from one Hub tunnel to the other should be seamless.
        set mode-cfg-allow-client-selector enable # Allows custom phase 2 selectors to be used
        set remote-fw <public ipv4 for hub-isp1-p1>
        set psksecret <pwd> # If using psk this should match what is set on the Hub above
        [set certificate <signature>] # If using certificates for authentication instead of PSK
        set network-overlay enable
        [set idle-timeout enable] # Not sure what this does just yet
        [set dpd-retryinterval 2] # Shortening the DPD health check when tunnel is idle
        [set dpd-retrycount 2]
    next
    edit "spoke-isp2-p1"
        set typic static
        set interface "wan2"
        set ike-version 2
        set peertype any
        set net-device enable # The spoke side may be behind a NAT device
        set network-id 2
        set localid "region1-spoke1"
        set proposal aes256-sha384 # NSA recommnended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set add-route enable # This is the default setting so will only show in full. We want to receive routes as part of IKE process.
        set dpd on-idle
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set auto-discovery-receiver enable # This will allow the spoke to learn ADVPN shortcuts to other spokes sent via the Hub
        [set auto-discovery-shortcuts dependent] # This controls whether the Spoke shortcuts should be deleted if the Hub tunnel goes down. If you are also using `tcp-session-without-syn enable` then the failover from one Hub tunnel to the other should be seamless.
        set mode-cfg-allow-client-selector enable # Allows custom phase 2 selectors to be used
        set remote-fw <public ipv4 for hub-isp2-p1>
        set psksecret <pwd> # If using psk this should match what is set on the Hub above
        [set certificate <signature>] # If using certificates for authentication instead of PSK
        set network-overlay enable
        [set idle-timeout enable] # Not sure what this does just yet
        [set dpd-retryinterval 2] # Shortening the DPD health check when tunnel is idle
        [set dpd-retrycount 2]
    next
end

config vpn ipsec phase2-interface
    edit "spoke-isp1-p2"
        set phase1name "spoke-isp1-p1"
        set proposal aes256-sha384 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set auto-negotiate enable
        set src-addr-type name
        set dst-addr-type name
        set src-name "r1-s1-lan" # This address group has to exist before referencing it. The object then gets advertised to Hub and other Spokes.
        set dst-name "all"
    next
    edit "spoke-isp2-p2"
        set phase1name "spoke-isp2-p1"
        set proposal aes256-sha384 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set auto-negotiate enable
        set src-addr-type name
        set dst-addr-type name
        set src-name "r1-s1-lan" # This address group has to exist before referencing it. The object then gets advertised to Hub and other Spokes.
        set dst-name "all"
    next
end
```

### VPN Interfaces

The tunnel interfaces won't display their assigned IP address in the GUI or when running the `get system interface` command, but will show when running the `diag address list` command and in other debugging commands.

```ruby
config system interface
    edit "spoke-isp1-p1"
        set type tunnel
        set allowaccess ping # ADVPN shortcut paths will open child-health checks bound to these interfaces dynamically
    next
    edit "spoke-isp2-p1"
        set type tunnel
        set allowaccess ping # ADVPN shortcut paths will open child-health checks bound to these interfaces dynamically
    next
    edit "spoke-loopback"
        set vdom "root" # This has to be entered or else it will not take
        set type loopback
        set ip 10.0.1.1 255.255.255.0 # This should be in the network range advertised by the Hub via BGP
        set allowaccess ping
    next
end
```

### SD-WAN Configuration

* *All existing configuration referencing an interface (or member interface in a zone) has to be removed before assigning the member interface*

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
        edit "hub_icmp"
            set server "10.10.255.1"
            set protocol icmp
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
                edit "hub_icmp"
                    set id 1
                next
            end
            set priority-members 3 4
        next
    end
end
```

### Static Routes

Default route to 0.0.0.0/0 includes both underlay interfaces and overlay interfaces via zones. In order for SD-WAN rules to direct traffic, there has to be an existing route (or routes) in the active routing table (`get router info routing-table all`)

```ruby
config router static
    edit 1
        set status enable
        set dst 0.0.0.0 0.0.0.0
        set distance 1
        set comment "default static route for internet access"
        set sdwan-zone "underlay-dia" "overlay-vpn" # You can also specify individual interface members instead of the sdwan-zone shortcut
    next
end
```

### BGP Routing

```ruby
config router bgp
    set as 65000
    set router-id <ipv4> # Best practice to set this; by default will take the highest loopback address if not defined
    set ibgp-multipath enable
    set network-import-check disable # By default this was enabled; just want a single route
    set graceful-restart enable
    config neighbor
        edit "169.254.1.253"
            set advertisement-interval 1
            set link-down-failover enable
            set remote-as 65000
        next
        edit "169.254.2.253"
            set advertisement-interval 1
            set link-down-failover enable
            set remote-as 65000
        next
    end
    config network
        edit 1
            set prefix 10.0.1.0 255.255.255.0 # Whatver spoke subnets that you want to advertise; not a summary address or default-route
        next
        edit 2
            set prefix <ipv4 network> # Whatver spoke subnets that you want to advertise; not a summary address or default-route
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
