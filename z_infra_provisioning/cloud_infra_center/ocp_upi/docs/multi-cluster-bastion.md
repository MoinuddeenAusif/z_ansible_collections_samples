## Configure another cluster's DNS and HAProxy on the same bastion node

After the bastion was configured by our Ansible scripts, named and HAProxy were installed. If you want to use the same node as another cluster's bation node, you need to add the second configuration manually.

### named

We use named as the DNS server, its configuration was stored as /etc/named.conf and /var/named/xxxx.zone

1. Add the second zone file in /var/named/, here is an example file `openshift.second.com.zone`:

```
$TTL 900
  
@                     IN SOA bastion.openshift.second.com. hostmaster.openshift.second.com. (
                        2019062002 1D 1H 1W 3H
                      )
                      IN NS bastion.openshift.second.com.

bastion               IN A 172.26.105.100
api                   IN A 172.26.105.100
api-int               IN A 172.26.105.100
apps                  IN A 172.26.105.100
*.apps                IN A 172.26.105.100

bootstrap           IN A 172.26.105.210

master-0              IN A 172.26.105.200
master-1              IN A 172.26.105.207
master-2              IN A 172.26.105.202

worker-2e582              IN A 172.26.105.208
worker-5e74f              IN A 172.26.105.209

etcd-0              IN A 172.26.105.200
etcd-1              IN A 172.26.105.207
etcd-2              IN A 172.26.105.202

_etcd-server-ssl._tcp IN SRV 0 10 2380 etcd-0.openshift.second.com.
                      IN SRV 0 10 2380 etcd-1.openshift.second.com.
                      IN SRV 0 10 2380 etcd-2.openshift.second.com.
```

You need to change the `openshift.second.com` to the second cluster domain name, and change servers' IPs and worker nodes' names.

2. Correct the zone file's owner

```
chown root:named openshift.second.com.zone
```

3. Add record in /etc/named.conf

Add below configuration to /etc/named.conf:

```
zone "openshift.second.com" {
        type master;
        file "openshift.second.com.zone";
        allow-query { any; };
        allow-transfer { none; };
        allow-update { none; };
};
```

4. Restart the named-chroot service

```
systemctl restart named-chroot
```

### HAProxy

HAProxy configuration was stored as /etc/haproxy/haproxy.cfg 

1. Add the second cluster configuration to /etc/haproxy/haproxy.cfg

You can add the second cluster configuration in /etc/haproxy/haproxy.cfg directly after the first cluster, here is an haproxy configuration file example:

```
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option                  forwardfor except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

# the first cluster configuration 
frontend ocp4-kubernetes-api-server
   mode tcp
   option tcplog
   bind api.openshift.first.com:6443
   bind api-int.openshift.first.com:6443
   default_backend ocp4-kubernetes-api-server

frontend ocp4-machine-config-server
   mode tcp
   option tcplog
   bind api.openshift.first.com:22623
   bind api-int.openshift.first.com:22623
   default_backend ocp4-machine-config-server

frontend ocp4-router-http
   mode tcp
   option tcplog
   bind apps.openshift.first.com:80
   default_backend ocp4-router-http

frontend ocp4-router-https
   mode tcp
   option tcplog
   bind apps.openshift.first.com:443
   default_backend ocp4-router-https

backend ocp4-kubernetes-api-server
   mode tcp
   balance source
   server bootstrap bootstrap.openshift.first.com:6443 check
   server master-0 master-0.openshift.first.com:6443 check
   server master-1 master-1.openshift.first.com:6443 check
   server master-2 master-2.openshift.first.com:6443 check

backend ocp4-machine-config-server
   mode tcp
   balance source
   server bootstrap bootstrap.openshift.first.com:22623 check
   server master-0 master-0.openshift.first.com:22623 check
   server master-1 master-1.openshift.first.com:22623 check
   server master-2 master-2.openshift.first.com:22623 check

backend ocp4-router-http
   mode tcp
         server worker-34de5 worker-34de5.openshift.first.com:80 check
         server worker-4fe2f worker-4fe2f.openshift.first.com:80 check

backend ocp4-router-https
   mode tcp
         server worker-34de5 worker-34de5.openshift.first.com:443 check
         server worker-4fe2f worker-4fe2f.openshift.first.com:443 check

# the second cluster configuration 
frontend ocp4-kubernetes-api-server-second
   mode tcp
   option tcplog
   bind api.openshift.second.com:6443
   bind api-int.openshift.second.com:6443
   default_backend ocp4-kubernetes-api-server-second

frontend ocp4-machine-config-server-second
   mode tcp
   option tcplog
   bind api.openshift.second.com:22623
   bind api-int.openshift.second.com:22623
   default_backend ocp4-machine-config-server-second

frontend ocp4-router-http-second
   mode tcp
   option tcplog
   bind apps.openshift.second.com:80
   default_backend ocp4-router-http-second

frontend ocp4-router-https-seconds
   mode tcp
   option tcplog
   bind apps.openshift.second.com:443
   default_backend ocp4-router-https-seconds

backend ocp4-kubernetes-api-server-second
   mode tcp
   balance source
   server bootstrap bootstrap.openshift.second.com:6443 check
   server master-0 master-0.openshift.second.com:6443 check
   server master-1 master-1.openshift.second.com:6443 check
   server master-2 master-2.openshift.second.com:6443 check

backend ocp4-machine-config-server-second
   mode tcp
   balance source
   server bootstrap bootstrap.openshift.second.com:22623 check
   server master-0 master-0.openshift.second.com:22623 check
   server master-1 master-1.openshift.second.com:22623 check
   server master-2 master-2.openshift.second.com:22623 check

backend ocp4-router-http-second
   mode tcp
         server worker-2e582 worker-2e582.openshift.second.com:80 check
         server worker-5e74f worker-5e74f.openshift.second.com:80 check

backend ocp4-router-https-seconds
   mode tcp
         server worker-2e582 worker-2e582.openshift.second.com:443 check
         server worker-5e74f worker-5e74f.openshift.second.com:443 check
```

**Note**: The second cluster's frontend and backend names can not be the same as first cluster ones.

2. Restart HAProxy

```
systemctl restrat HAProxy
```

3. Test if DNS and HAProxy works

Lookup the domain name and it can be resloved.

```
$ nslookup api.openshift.second.com
Server:		172.26.105.100
Address:	172.26.105.100#53

Name:	api.openshift.second.com
Address: 172.26.105.100
```

Curl the domain name and it returns HTTP status result.

```
$ curl api-int.openshift.second.com
curl: (52) Empty reply from server
```
