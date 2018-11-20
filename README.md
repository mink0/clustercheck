# Clustercheck

Lightweight health checks boilerplate for cluster nodes monitoring using `bash` and `xinetd`.

Below is a sample configuration for HAProxy. The node will be removed from cluster if healthcheck fails.
Cluster node health is checked via HTTP requests through `9200/tcp` socket.

`/etc/haproxy/haproxy.cfg`

    ...
    listen myapp 0.0.0.0:8080
      option httpchk
      mode tcp
        server node1 10.0.0.1:80 check fall 3 rise 2 port 9200
        server node2 10.0.0.2:80 check fall 3 rise 2 port 9200

The `clustercheck` is a simple shell script that run health checks on every incoming request to the port `9200/tcp`. If the health checks are passed, it will respond with `HTTP code 200 (OK)`, otherwise a `HTTP error 503 (Service Unavailable)` message is returned. Any health checks could be added to the script body.

## Install with Ansible

        cd ./ansible
        ansible-playbook -i 'srv.domain.com,' ./clustercheck.yml -K

## Manual setup with xinetd

This setup will create a process that listens on TCP port 9200 using `xinetd`. This process uses the `clustercheck` script from this repository to report the status of the node.

- Install `xinetd`

- Copy the clustercheck from the repository to a location (`/usr/bin/` in the example below) and make it executable

- Add the following service to `xinetd` (make sure to match your location of the script with the `server` entry):

`/etc/xinetd.d/clustercheck`

    service clustercheck
    {
        server = /usr/bin/clustercheck
        port = 9200
        wait = no
        type = UNLISTED
        user = nobody
        flags = REUSE
        socket_type = stream
        log_on_failure += USERID
        only_from = 0.0.0.0/0
        per_source = UNLIMITED
    }

- Restart xinetd

- (Optional) Open firewall port 9200/tcp for external checks

- (Optional) Add custom health check logic to the `/usr/bin/clustercheck`

Clustercheck will now listen on port `9200/tcp`, you could run `curl http://0:9200` from the node to check everything works properly.

## Manual setup shell script only

If you do not want to use the setup with xinetd, you can also execute `clustercheck` on the command line and check for the return value.

In case of a healthy node:

    $ /usr/bin/clustercheck > /dev/null
    $ echo $?
    0

In case of an unhealthy node:

    $ /usr/bin/clustercheck > /dev/null
    $ echo $?
    1

You can use this return value with monitoring tools like `Zabbix` or `Zenoss`.

## Command line usage

The clustercheck script accepts this arguments:

    clustercheck <enable|disable>

- **disable**: Force clustercheck to always return 503 and the node is being removed from the cluster (maintenance mode).
- **enable**: Return clustercheck back to normal behaviour.

## Manually removing node from cluster

By touching `/var/tmp/node.disable` file system administrator may force clustercheck to return `503` regardless to the actual state of the node. This is useful when the node is being put into maintenance mode.
