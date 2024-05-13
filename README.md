# FreeRadius Server Docker Image

FreeRadius Docker based on Ubuntu 20.04 LTS, optimized for ISP's.

# Environment Variables

| ENV                         | VALUE              | TYPE     |
|-----------------------------|--------------------|----------|
| MYSQL_HOST                  | mysql              | required |
| MYSQL_PORT                  | 3306               | required |
| MYSQL_DATABASE              | radius             | required |
| MYSQL_PASSWORD              | radpass            | required |
| MYSQL_USER                  | radius             | required |
| MYSQL_INIT                  | true               | optional |
| DEFAULT_CLIENT_SECRET       | testing123         | optional |
| PPP_VAN_JACOBSON_TCP_IP     | false              | optional |
| TZ                          | Europe/Bratislava  | optional |
| EAP_USE_TUNNELED_REPLY      | true               | optional |
| COA_RELAY_ENABLE            | true               | optional |
| COA_RELAY_INTERFACE         | eth0 (SWARM = eth1)| optional |
| COA_RELAY_PACKET_DST_PORT   | 3799               | optional |
| COA_RELAY_NAS_IP_n          | 127.0.0.1          | optional |
| COA_RELAY_NAS_PORT_n        | 3799               | optional |
| COA_RELAY_NAS_SECRET_n      | secret             | optional |
| STATUS_ENABLE               | true               | optional |
| STATUS_INTERFACE            | eth0 (SWARM = eth1)| optional |
| STATUS_CLIENT               | exporter           | optional |
| STATUS_SECRET               | adminsecret1       | optional |

## STATUS_ENABLE = 'true'

```diff
server status {
        listen {
                #  ONLY Status-Server is allowed to this port.
                #  ALL other packets are ignored.
                type = status

-               ipaddr = 127.0.0.1
+               ipaddr = {$RADIUS_CONTAINER_IP}
                port = 18121
        }

        #
        #  We recommend that you list ONLY management clients here.
        #  i.e. NOT your NASes or Access Points, and for an ISP,
        #  DEFINITELY not any RADIUS servers that are proxying packets
        #  to you.
        #
        #  If you do NOT list a client here, then any client that is
        #  globally defined (i.e. all of them) will be able to query
        #  these statistics.
        #
        #  Do you really want your partners seeing the internal details
        #  of what your RADIUS server is doing?
        #
-       client admin {
+       client {$STATUS_CLIENT} {
-               ipaddr = 127.0.0.1
+               ipaddr = 0.0.0.0
-               secret = adminsecret
+               secret = {$STATUS_SECRET}
        }

        #
        #  Simple authorize section.  The "Autz-Type Status-Server"
        #  section will work here, too.  See "raddb/sites-available/default".
        authorize {
                ok

                # respond to the Status-Server request.
                Autz-Type Status-Server {
                        ok
                }
        }
}
```

## COA_RELAY_ENABLE = 'true'

### Documentation

This virtual server simplifies the process of sending CoA-Request or
Disconnect-Request packets to a NAS.

This virtual server will receive CoA-Request or Disconnect-Request
packets that contain *minimal* identifying information.  e.g. Just
a User-Name, or maybe just an Acct-Session-Id attribute.  It will
look up that information in a database in order to find the rest of
the session data.  e.g. NAS-IP-Address, NAS-Identifier, NAS-Port,
etc.  That information will be added to the packet, which will then
be sent to the NAS.

This process is useful because NASes require the CoA packets to
contain "session identification" attributes in order to to do CoA
or Disconnect.  If the attributes aren't in the packet, then the
NAS will NAK the request.  This NAK happens even if you ask to
disconnect "User-Name = bob", and there is only one session with a
"bob" active.

Using this virtual server makes the CoA or Disconnect process
easier.  Just tell FreeRADIUS to disconnect "User-Name = bob", and
FreeRADIUS will take care of adding the "session identification"
attributes.

The process is as follows:
  - A CoA/Disconnect-Request is received by FreeRADIUS.
  - The radacct table is searched for active sessions that match each of
    the provided identifier attributes: User-Name, Acct-Session-Id. The
    search returns the owning NAS and Acct-Unique-Id for the matching
    session/s.
  - The original CoA/Disconnect-Request content is written to a detail file
    with custom attributes representing the NAS and Acct-Session-Id.
  - A detail reader follows the file and originates CoA/Disconenct-Requests
    containing the original content, relayed to the corresponding NAS for
    each session using the custom attributes.
This simplifies scripting directly against a set of NAS devices since a
script need only send a single CoA/Disconnect to FreeRADIUS which will
then:
  - Lookup all active sessions belonging to a user, in the case that only a
    User-Name attribute is provided in the request
  - Handle routing of the request to the correct NAS, in the case of a
    multi-NAS setup

For example, to disconnect a specific session:

```console
echo 'Acct-Session-Id = "769df3 312343"' | \
    radclient 127.0.0.1 disconnect testing123
```

To perform a CoA update of all active sessions belonging to a user:

```console
cat <<EOF | radclient 127.0.0.1 coa testing123
User-Name = bob
Cisco-AVPair = "subscriber:sub-qos-policy-out=q_out_uncapped"
EOF
```

In addition to configuring and activating this site, a detail
writer module must be configured in mods-enabled:

```console
detail detail_coa {
    filename = ${radacctdir}/detail_coa
    escape_filenames = no
    permissions = 0600
    header = "%t"
    locking = yes
}
```

### Radclient examples


```console
echo "User-Name = pppoe@example.com,User-Password = simulate" | \
    radclient -x -s '127.0.0.1' auth testing123
```

This command is sending an authentication request to a RADIUS server.

- `echo "User-Name = pppoe@example.com,User-Password = simulate"`: This part is creating a string that contains the RADIUS attributes for the username and password.
- `radclient -x -s '127.0.0.1' auth testing123`: This part is sending the attributes to the RADIUS server at IP address `127.0.0.1` using the `auth` command (which sends an Access-Request packet). The `-x` option enables debug mode, and `-s` prints out some statistics. `testing123` is the shared secret expected by the RADIUS server.

```console
echo "User-Name = pppoe@example.com" | \
    radclient -x -s '127.0.0.1' disconnect testing123
```

This command is sending a disconnect request to a RADIUS server.

- `echo 'User-Name = "pppoe@example.com"'`: This part is creating a string that contains the RADIUS attribute for the username.
- `radclient -X -s '127.0.0.1' disconnect testing123`: This part is sending the attribute to the RADIUS server at IP address `127.0.0.1` using the `disconnect` command (which sends a Disconnect-Request packet). The `-X` option enables more detailed debug mode, and `-s` prints out some statistics. `testing123` is the shared secret expected by the RADIUS server.

```console
echo "User-Name = pppoe@example.com,Framed-IP-Address = 100.64.0.107" | \
    radclient -x -s '127.0.0.1' coa testing123
```

This command is sending a CoA (Change of Authorization) request to a RADIUS server.

- `echo "User-Name = pppoe@example.com,Framed-IP-Address = 100.64.0.107"`: This part is creating a string that contains the RADIUS attributes for the username and the framed IP address.
- `radclient -X -s '127.0.0.1' coa testing123`: This part is sending the attributes to the RADIUS server at IP address `127.0.0.1` using the `coa` command (which sends a CoA-Request packet). The `-X` option enables more detailed debug mode, and `-s` prints out some statistics. `testing123` is the shared secret expected by the RADIUS server.

A CoA request is used to change the attributes of a user session after it has been authenticated. In this case, it's specifying a username and a framed IP address. The server will use these attributes to find the session and apply the changes.


```console
echo 'User-Name = "pppoe@example.com",Cisco-Service-Info = "QD;629145600;117964800;235929600;U;62914560;11796480;23592960"' | \
    radclient -x -s '127.0.0.1' coa testing123
```

```console
echo 'User-Name = "pppoe@example.com",Cisco-Service-Info = "QD;1048576000;196608000;393216000;U;104857600;19660800;39321600"' | \
    radclient -x -s '127.0.0.1' coa testing123
```

## PPP_VAN_JACOBSON_TCP_IP = 'false'

See [ASR 1002-X PPPoE problem with Virtual-Access sub-interfaces](https://community.cisco.com/t5/other-service-provider-subjects/asr-1002-x-pppoe-problem-with-virtual-access-sub-interfaces/td-p/2665369).

```diff
DEFAULT Framed-Protocol == PPP
-        Framed-Protocol = PPP,
-        Framed-Compression = Van-Jacobson-TCP-IP
+        Framed-Protocol = PPP
+        #Framed-Compression = Van-Jacobson-TCP-IP
```
