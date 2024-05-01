# SD-WAN with BGP on Loopback (Dual-Hub Region)

_largely adapted from (SD-Deployment for MSSPs Guides)[https://docs.fortinet.com/document/fortigate/7.2.0/sd-wan-deployment-for-mssps/590594/bgp-on-loopback-dual-hub-region]_

## Spoke

### Loopback Interface

```ruby
config system interface
    edit "spoke-loopback"
        set vdom "root"
        set type loopback
        set ip 10.200.1.1/32
        set allowaccess ping
    next
end
```

### Configure Unique Location-ID

```ruby
config system settings
    set location-id 10.200.1.1 # best practice is to use the loopback address
end
```

### VPN Settings specific for injecting static route to reach the loopback on all p1-interfaces towards the hub

```ruby
config vpn ipsec phase1-interface
    edit "H1_ISP1"
        set exchange-ip-addr4 10.200.1.1 # Causes IKE to inject a static route towards the spoke loopback interface (where BGP will be bound)
    next
    edit "H1_MPLS"
        set exchange-ip-addr4 10.200.1.1
    next
    edit "H2_ISP1"
        set exchange-ip-addr4 10.200.1.1
    next
    edit "H2_MPLS"
        set exchange-ip-addr4 10.200.1.1
    next
end
```

### Configure Route-Maps to Apply Tag per Hub

```ruby
config router route-map
    edit "H1_TAG"
        config rule
            edit 1
            set set-tag 1
        next
    end
    next
    edit "H2_TAG"
        config rule
            edit 1
            set set-tag 2
        next
    end
    next
end
```

### Configure BGP

* Single neighbor per Hub (using the Hub's loopback interface to form adjacency)
* Apply route-maps to tag ingress route received from each hub with a specific tag
* No ADD-PATH settings are needed as compared to BGP per Overlay configuration
* Must enable `set tag-resolve-mode merge`
* Must enbale `set recursive-next-hop enable`

```ruby
config router bgp
    set as 65001
    set router-id 10.200.1.1 # Best practice to set this to loopback address; by default will take the highest loopback address if not defined
    set keepalive-timer 15 # Adjust BGP timers to best fit your circuits and environment
    set holdtime-timer 45
    set ibgp-multipath enable
    set recursive-next-hop enable # Crucial settings for when using iBGP
    set tag-resolve-mode merge # Merge routes received from Hub via tags
    config neighbor
        edit 10.200.1.253
            set soft-reconfiguration enable
            set advertisement-interval 1 # Adjust BGP timers to best fit your circuits and environment
            set interface "Lo"
            set update-source "Lo"
            set connect-timer 1 # Adjust BGP timers to best fit your circuits and environment
            set remote-as 65001
            set route-map-in "H1_TAG"
        next
        edit 10.200.1.254
            set soft-reconfiguration enable
            set advertisement-interval 1 # Adjust BGP timers to best fit your circuits and environment
            set interface "Lo"
            set update-source "Lo"
            set connect-timer 1 # Adjust BGP timers to best fit your circuits and environment
            set remote-as 65001
            set route-map-in "H2_TAG"
        next
    end
    config network
        edit 1
            set prefix 10.0.1.0/24 # Whatever spoke subnets that you want to advertise; not a summary address or default-route
        next
    end
end
```

### SD-WAN Member Configuration

* Set the source to the loopback address on each member, so that health-checks are always sourced from the loopback

```ruby
config system sdwan
    config members
        edit 2
            set source 10.200.1.1 # Specify the loopback address so that health-checks as always sourced from the loopback interface
        next
        edit 3
            set source 10.200.1.1 # Specify the loopback address so that health-checks as always sourced from the loopback interface
        next
    end
end
```

### Firewall Policy



Embedded SLA information from spokes, called __Remote Health Probing__