# SD-WAN Hub-Spoke with BGP Community Tags

This document demonstrates how to use BGP community tags with Hub-Spoke SD-WAN connections to influence the hub's traffic path to the spoke. This alters the basic Hub-Spoke configuration where the Hub previously did not use SD-WAN, instead relying entirely on staying "sticky" for the return path to the spoke. The address objects, groups, and interfaces are based on the SD-WAN with _IKE\_SA_ Hub-Spoke document_.

## Spoke Configuration

### Route Maps and Access Lists

```ruby
config router access-list
    edit "acl-r1-s1"
        config rule
            edit 1
                set prefix 10.0.1.0/24
            next
        end
    next
end

config router route-map
    edit "spoke-isp1_rm"
        config rule
            edit 1
                set match-ip-address "acl-r1-s1"
                set set-community "65000:1"
            next
        end
    next
    edit "spoke-isp2_rm"
        config rule
            edit 1
                set match-ip-address "acl-r1-s1"
                set set-community "65000:2"
            next
        end
    next
    edit "sla-failed_rm"
        config rule
            edit 1
                set match-ip-address "acl-r1-s1"
                set set-community "65000:5000"
            next
        end
    next
end
```

### BGP Routing with Community Tags

```ruby
config router bgp
    config neighbor
        edit "169.254.1.253"
            set route-map-out "sla-failed_rm"
            set route-map-out-preferable "spoke-isp1_rm"
        next
        edit "169.254.2.253"
            set route-map-out "sla-failed_rm"
            set route-map-out-preferable "spoke-isp2_rm"
        next
    end
end
```

### SD-WAN Neighbor for BGP According to SLA

```ruby
config sys sdwan
    config neighbor
        edit "169.254.1.253"
            set member 3
            set health-check "hub_twamp"
            set sla-id 1
        next
        edit "169.254.2.253"
            set member 4
            set health-check "hub_twamp"
            set sla-id 1
        next
    end
end
```

## Hub Configuration

### Community Lists and Route Maps

```ruby
config router community-list
    edit "65000:1_cl"
        config rule
            edit 1
                set action permit
                set match "65000:1"
            next
        end
    next
    edit "65000:2_cl"
        config rule
            edit 1
                set action permit
                set match "65000:2"
            next
        end
    next
    edit "65000:5000_cl"
        config rule
            edit 1
                set action permit
                set match "65000:5000"
            next
        end
    next
end

config router route-map
    edit "advpn-spoke_rm-in"
        config rule
            edit 1
                set match-community "65000:1_cl"
                set set-route-tag 1
            next
            edit 2
                set match-community "65000:2_cl"
                set set-route-tag 2
            next
            edit 3
                set match-community "65000:5000_cl"
                set set-route-tag 5000
            next
        end
    next
end
```

### BGP Routing with Community Tags

```ruby
config router bgp
    config neighbor-group
        edit "spoke-advpn"
            set route-map-in "advpn-spoke_rm-in"
        next
    end
end
```

### SD-WAN Neighbor for BGP According to SLA

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
    config service
        edit 1
            set name "hub_to_spoke_isp1"
            set route-tag 1 # Remember, if there is no matching route-tag, this rule isn't a match and next service rule gets evaluated
            set priority-members 1 # As long as route-tag matches, this interface will always be used
        next
        edit 2
            set name "hub_to_spoke_isp2"
            set route-tag 2 # Remember, if there is no matching route-tag, this rule isn't a match and next service rule gets evaluated
            set priority-members 2 # As long as route-tag matches, this interface will always be used
        next
    end
end
```
