# Logspout Charm

This charm is protoware - not suitable for anything other than exploring so far.



Deployment
----------


You can deploy this charm from my namespace in the charm store:

    juju deploy trusty/docker
    juju deploy cs:~lazypower/trusty/logspout
    juju deploy cs:~evarlast/trusty/logstash
    juju deploy trusty/elasticsearch
    juju deploy trusty/kibana
    juju add-relation logspout:docker-host docker
    juju add-relation logspout logstash
    juju add-relation logstash elasticsearch
    juju add-relation elasticsearch kibana:rest


With this formation, you should get something resembling the following:

![](http://i.imgur.com/t8vJwSJ.png)


From here, simply expose your kibana endpoint, and browse to the public-address
and start building your logstash dashboard. You'll notice that all the logs
coming from your docker containers are now being indexed in elasticsearch
in a raw JSON format.


Caveats
-------

You'll need to enable your own UDP parsing schema for elasticsearch

I put the following in `/etc/logstash/conf.d/input-docker.conf`

    input {
      tcp {
        port => 5000
        type => syslog
      }
      udp {
        port => 5000
        type => syslog
      }
    }

    filter {
      if [type] == "syslog" {
        grok {
          match => { "message" => "%{SYSLOG5424PRI}%{NONNEGINT:ver} +(?:%{TIMESTAMP_ISO8601:ts}|-) +(?:%{HOSTNAME:containerid}|-) +(?:%{NOTSPACE:containername}|-) +(?:%{NOTSPACE:proc}|-) +(?:%{WORD:msgid}|-) +(?:%{SYSLOG5424SD:sd}|-|) +%{GREEDYDATA:msg}" }
        }
        syslog_pri { }
        date {
          match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
        if !("_grokparsefailure" in [tags]) {
          mutate {
            replace => [ "@source_host", "%{syslog_hostname}" ]
            replace => [ "@message", "%{syslog_message}" ]
          }
        }
        mutate {
          remove_field => [ "syslog_hostname", "syslog_message", "syslog_timestamp" ]
        }
      }
    }

Currently, the logs are a bit scattered, and dont parse well. If you are a logstash regexer master, I want to talk to you :)

But you can still create searches and get meaningful metrics from the logs.

![](http://i.imgur.com/RhdnRvd.png)


Todo:
------

- The ignored containers config option currently no-op's and needs additional logic when searching for containers / naming schema / ids / etc.
- Scaling to multiple-hosts has not been tested
- Work with multiple-logstash-input points has not been tested


Feedback:
--------

Bugs, comments, pull-requests are more than welcome
