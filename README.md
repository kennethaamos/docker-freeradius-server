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
| EAP_USE_TUNNELED_REPLY      | true               | required |
| COA_RELAY_ENABLE            | true               | optional |
| STATUS_ENABLE               | true               | optional |
| STATUS_CLIENT               | exporter           | optional |
| STATUS_SECRET               | adminsecret1       | optional |
| STATUS_INTERFACE            | eth0 (SWARM = eth1)| optional |
| PPP_VAN_JACOBSON_TCP_IP     | false              | optional |
| TZ                          | Europe/Bratislava  | optional |

## STATUS_ENABLE = true

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

## STATUS_ENABLE = true

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

## PPP_VAN_JACOBSON_TCP_IP = false

See [ASR 1002-X PPPoE problem with Virtual-Access sub-interfaces](https://community.cisco.com/t5/other-service-provider-subjects/asr-1002-x-pppoe-problem-with-virtual-access-sub-interfaces/td-p/2665369).

```diff
DEFAULT Framed-Protocol == PPP
-        Framed-Protocol = PPP,
-        Framed-Compression = Van-Jacobson-TCP-IP
+        Framed-Protocol = PPP
+        #Framed-Compression = Van-Jacobson-TCP-IP
```
