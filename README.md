[![Code Climate](https://codeclimate.com/github/kkirsche/net-netconf/badges/gpa.svg)](https://codeclimate.com/github/kkirsche/net-netconf)

# NetConf

## Description:

Device management using the NetConf protocol as specified in [RFC4741](http://tools.ietf.org/html/rfc4741)
and [RFC6241](http://tools.ietf.org/html/rfc6241).

## Features:

* Extensible protocol transport framework for SSH and non-SSH
  * SSH transport using [Net::SSH::Multi](https://github.com/jamis/net-ssh-multi)
  * Telnet transport using `Net::Telnet` (Ruby Library)
  * Serial transport using [Ruby/SerialPort](http://ruby-serialport.rubyforge.org/)
* NETCONF Standard RPCs
  * `get-config`, `edit-config`
  * `lock`, `unlock`
  * `validate`, `discard-changes`
* Flexible RPC mechanism
  * `Netconf::RPC::Builder` to metaprogram RPCs
  * Vendor extension framework for custom RPCs
* XML processing using [Nokogiri](http://nokogiri.org)

## Synopsis:

```ruby
require 'net/netconf'

# create the options hash for the new NETCONF session.  If you are
# using ssh-agent, then omit the :password

login = { target: 'vsrx', username: 'root', password: 'Amnesia' }

# provide a block and the session will open, execute, and close

Netconf::SSH.new(login){ |dev|

  # perform the RPC command:
  # <rpc>
  #    <get-chassis-inventory/>
  # </rpc>

  inv = dev.rpc.get_chassis_inventory

  # The response is in Nokogiri XML format for easy processing ...

  puts 'Chassis: ' + inv.xpath('chassis/description').text
  puts 'Chassis Serial-Number: ' + inv.xpath('chassis/serial-number').text
}
```

Alternative explicity open, execute RPCs, and close

```ruby
require 'net/netconf'

dev = Netconf::SSH.new(login)
dev.open

inv = dev.rpc.get_chassis_inventory

puts 'Chassis: ' + inv.xpath('chassis/description').text
puts 'Chassis Serial-Number: ' + inv.xpath('chassis/serial-number').text

dev.close
```

## Using NetConf:

### Remote Procedure Calls (RPCs):

Each Netconf session provides a readable instance variable - __rpc__. This is used to execute Remote Procedure Calls (RPCs).
The @rpc will include the NETCONF standard RPCs, any vendor specific extension, as well as the ability
to metaprogram new onces via method_missing.

Here are some examples to illustrate the metaprogamming:

Without any parameters, the RPC is created by swapping underscores (`_`) to hyphens (`-`):

```ruby
 require 'net/netconf'

 dev.rpc.get_chassis_inventory

 # <rpc>
 #    <get-chassis-inventory/>
 # </rpc>
```

You can optionally provide RPC parameters as a hash:

```ruby
 dev.rpc.get_interface_information(interface_name: 'ge-0/0/0', terse: true)

 # <rpc>
 #    <get-interface-information>
 #       <interface-name>ge-0/0/0</interface-name>
 #       <terse/>
 #   </get-interface-information>
 # </rpc>
```

You can additionally supply attributes that get assigned to the toplevel element.  In this case
You must enclose the parameters hash to disambiquate it from the attributes hash, or declare
a variable for the parameters hash.

```ruby
 dev.rpc.get_interface_information({ interface_name: 'ge-0/0/0', terse: true },
                                   { format: 'text'})

 # <rpc>
 #    <get-interface-information format='text'>
 #       <interface-name>ge-0/0/0</interface-name>
 #       <terse/>
 #   </get-interface-information>
 # </rpc>
```

If you want to provide attributes, but no parameters, then:

```ruby
 dev.rpc.get_chassis_inventory(nil, format: 'text')

 # <rpc>
 #    <get-chassis-inventory format='text'/>
 # </rpc>
```

### Retrieving Configuration:

To retrieve configuration from a device, use the `get-config` RPC.  Here is an example, but you can find others in the __examples__ directory:

```ruby
 require 'net/netconf'


 login = { target: 'vsrx', username: 'root', password: 'Amnesia' }

 puts "Connecting to device: #{login[:target]}"

 Netconf::SSH.new(login){ |dev|
   puts 'Connected.'

   # ----------------------------------------------------------------------
   # retrieve the full config.  Default source is 'running'
   # Alternatively you can pass the source name as a string parameter
   # to #get_config

   puts 'Retrieving full config, please wait ... '
   cfgall = dev.rpc.get_config
   puts "Showing 'system' hierarchy ..."
   puts cfgall.xpath('configuration/system')     # JUNOS toplevel config element is <configuration>

   # ----------------------------------------------------------------------
   # specifying a filter as a block to get_config

   cfgsvc1 = dev.rpc.get_config do |x|
     x.configuration { x.system { x.services } }
   end

   puts 'Retrieved services as BLOCK:'
   cfgsvc1.xpath('//services/*').each { |s| puts s.name }

   # ----------------------------------------------------------------------
   # specifying a filter as a parameter to get_config

   filter = Nokogiri::XML::Builder.new do |x|
     x.configuration { x.system { x.services } }
   end

   cfgsvc2 = dev.rpc.get_config(filter)
   puts 'Retrieved services as PARAM:'
   cfgsvc2.xpath('//services/*').each { |s| puts s.name }

   cfgsvc3 = dev.rpc.get_config(filter)
   puts 'Retrieved services as PARAM, re-used filter'
   cfgsvc3.xpath('//services/*').each { |s| puts s.name }
 }
```

__Note:__ There is a JUNOS RPC, `get-configuration`, that provides Juniper specific extensions as well.

### Changing Configuration:

To retrieve configuration from a device, use the `edit-config` RPC.  Here is an example, but you can find
others in the __examples__ directory:

```ruby
  require 'net/netconf'

  login = { target: 'vsrx', username: 'root', password: 'Amnesia' }

  new_host_name = 'vsrx-abc'

  puts "Connecting to device: #{login[:target]}"

  Netconf::SSH.new(login){ |dev|
    puts 'Connected!'

    target = 'candidate'

    # JUNOS toplevel element is 'configuration'

    location = Nokogiri::XML::Builder.new do |x|
      x.configuration {
        x.system {
          x.location {
            x.building 'Main Campus, A'
            x.floor 5
            x.rack 27
          }
        }
      }
    end

    begin
      rsp = dev.rpc.lock target

      # --------------------------------------------------------------------
      # configuration as BLOCK

      rsp = dev.rpc.edit_config do |x|
        x.configuration {
          x.system {
            x.send(:'host-name', new_host_name )
          }
        }
      end

      # --------------------------------------------------------------------
      # configuration as PARAM

      rsp = dev.rpc.edit_config(location)

      rsp = dev.rpc.validate target
      rpc = dev.rpc.commit
      rpc = dev.rpc.unlock target

    rescue Netconf::LockError => e
      puts 'Lock error'
    rescue Netconf::EditError => e
      puts 'Edit error'
    rescue Netconf::ValidateError => e
      puts 'Validate error'
    rescue Netconf::CommitError => e
      puts 'Commit error'
    rescue Netconf::RpcError => e
      puts 'General RPC error'
    else
      puts 'Configuration Committed.'
    end
  }
```

__Note:__ There is a JUNOS RPC, `load-configuration`, that provides Juniper specific extensions as well.

## Original License:

(BSD 2)

Copyright (c) 2013, Juniper Networks

All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

(1) Redistributions of source code must retain the above copyright notice, this
list of conditions and the following disclaimer.

(2) Redistributions in binary form must reproduce the above copyright notice,
this list of conditions and the following disclaimer in the documentation
and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR
ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

The views and conclusions contained in the software and documentation are those
of the authors and should not be interpreted as representing official policies,
either expressed or implied, of Juniper Networks.

Original Authors:
* Jeremy Schulman, Juniper Networks
* Ankit Jain, Juniper Networks
