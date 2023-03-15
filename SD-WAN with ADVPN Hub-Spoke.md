# SD-WAN with ADVPN Hub-Spoke Within Single Region

*with vpn algoritihms recommendations from NSA PP-22-0266 Mar 2022 ver 1.0*

## Hub Configuration

### IPSEC Configuration

```ruby
config vpn ipsec phase1-interface
    edit "hub-isp1-p1"
        set type dynamic
        set interface "wan1"
        set peertype any
        set net-device disable # The hub is not beind a NAT device
        set proposal aes256-sha384 # NSA recommnended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set add-route disable # We do not want to add spoke routes to the hub
        set auto-discovery-sender enable # Hub is going to be sending spokes ADVPN shortcut info
        set dpd on-demand
        set ike-version 2
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set ipv4-start-ip 169.254.1.1 # The tunnel IP endpoint range doesn't need to be advertised in dynamic routing since it's a locally connected interface
        set ipv4-end-ip 169.254.1.250
        set ipv4-netmask 255.255.255.0
        set psksecret <psk>
        [set certificate <signature>]
        set network-overlay enable # This was in 4D doc, so questionable
    next
    edit "hub-isp2-p1"
        set type dynamic
        set interface "wan2"
        set peertype any
        set net-device disable # The hub is not beind a NAT device
        set proposal aes256-sha384 # NSA recommnended setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set add-route disable # We do not want to add spoke routes to the hub
        set auto-discovery-sender enable # Hub is going to be sending spokes ADVPN shortcut info
        set dpd on-demand
        set ike-version 2
        set mode-cfg enable # Purpose of this command and the following ipv4 commands is to auto-assign the remote p1 virtual interfaces ip
        set ipv4-start-ip 169.254.2.1
        set ipv4-end-ip 169.254.2.250
        set ipv4-netmask 255.255.255.0
        set psksecret <psk>
        [set certificate <signature>]
        set network-overlay enable # This was in 4D doc, so questionable
    next
end

config vpn ipsec phase2-interface
    edit "hub-isp1-p2"
        set phase1name "hub-isp1-p1"
        set proposal aes256-sha384 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
    next
    edit "hub-isp2-p2"
        set phase1name "hub-isp2-p1"
        set proposal aes256-sha384 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
        set dhgrp 20 16 # NSA recommneded setting. This has to match spokes and both sides must be capable of supporting it.
    next
end
```

### VPN Interfaces

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
        set ip <ipv4>
        set allowaccess ping probe-response # probe-response for TWAMP health check, may need to be bound to virtual p1 interface
        set type loopback
end
```

### BGP Routing

*iBGP is used between hub and spokes for SD-WAN*

```ruby
config router bgp
    set as 65000
    set ibgp-multipath enable
    *there should be a max multipath command here*
    set graceful-restart enable
    config neighbor-group
        edit "spoke-advpn"
            set link-down-failover enable
            set capability-graceful-restart enable
            set next-hop-self enable
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
            set prefix <ipv4 network> # Whatever data center subnets that you want to advertise; not summary addresses or default-route
        next
        edit 2
            set prefix <ipv4 network> # Whatever data center subnets that you want to advertise; not summary addresses or default-route
        next
    end
end
```

### FortiGate as TWAMP Server for SD-WAN Health Check

```ruby
config system probe-response
    set port 862
    set mode twamp
    set security-mode authentication # this may need to be adjusted if not doing control and just default test from spoke
    set password <pwd>
end
```

### Firewall Policies for SD-WAN

```ruby
config firewall address
    edit "spoke-tunnels"
        set subnet "169.254.1.0 255.255.255.0" "169.254.2.0 255.255.255.0"
    next
    edit "hub-subnets"
        set subnet <should match the bgp prefixes advertised to spokes>
    next
    edit "region-spokes"
        set subnet <should be all spoke subnets within region that could traverse thru hub>
    next
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
        set service "ALL_ICMP" "TWAMP" # May have to create a service for TWAMP is there's not a predefined one
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
        set service <svs to allow> # Also take into account any SD-WAN Services and Health Checks
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
        set service "all" # Spoke should already be controlling what they allow outbound
        set action accept
        set logtraffic all # May want to disable this as it could get very noisy and spokes may also be logging already
    next
end
```

## Spoke Configuration

### IPSEC Configuration
