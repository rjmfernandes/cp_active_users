# CP Active Users and Operations Auditing/Exclusion

- [CP Active Users and Operations Auditing/Exclusion](#cp-active-users-and-operations-auditingexclusion)
  - [Setup](#setup)
  - [Fetch Users per time and operation](#fetch-users-per-time-and-operation)
  - [Only the unique users from today](#only-the-unique-users-from-today)
  - [Operations Auditing/Exclusion](#operations-auditingexclusion)
  - [Cleanup](#cleanup)

## Setup

Setup demo environment as per https://docs.confluent.io/platform/current/security/authorization/rbac/cp-rbac-example.html (please check requirements https://docs.confluent.io/platform/current/security/authorization/rbac/cp-rbac-example.html#prerequisites).

```shell
git clone https://github.com/confluentinc/examples.git
cd examples
git checkout 7.7.1-post
cd security/rbac/scripts
./init.sh
./enable-rbac-broker.sh
```

Check user and password for cluster admin user:

```shell
cat /tmp/login.properties
```

Let's generate a command config file for our user/password MySystemAdmin/MySystemAdmin1:

```shell
echo '
sasl.mechanism=OAUTHBEARER
security.protocol=SASL_PLAINTEXT
sasl.login.callback.handler.class=io.confluent.kafka.clients.plugins.auth.token.TokenUserLoginCallbackHandler
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required username="MySystemAdmin" password="MySystemAdmin1" metadataServerUrls="http://localhost:8090";
'  > ../delta_configs/clientsma.properties.delta
```

You can also login with user/password MySystemAdmin/MySystemAdmin1:

```shell
confluent login --url http://localhost:8090
```

And describe audit-log config:

```shell
confluent audit-log config describe
```

## Fetch Users per time and operation

Now let's consume from our audit log topic `confluent-audit-log-events`:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events --max-messages 10 | jq
```

And after filter out what we are interested in:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events | jq '.time + "," + .data.authenticationInfo.principal + "," + .data.authorizationInfo.operation'
```

## Only the unique users from today

Run first:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events > time_users.txt
```

After the file time_users.txt stabilizes in size you can cancel the job. You can check that the file stabilizes in size by running consecutively in a separate shell:

```shell
wc -l time_users.txt
```

Then run:

```shell
cat time_users.txt | jq '.time + "," + .data.authenticationInfo.principal' | grep 'User:' | grep -n -e $(date +"%Y-%m-%dT") | sed s/'.*,User:'//g | sed s/'\"$'//g | sort | uniq
```

This will give you the list of active users today at least from the point of view of your audited operations.

## Operations Auditing/Exclusion

Run:

```shell
confluent audit-log config describe > config.json
```

You should have for `config.json` something like this:

```json
{
  "destinations": {
    "topics": {
      "confluent-audit-log-events": {
        "retention_ms": 7776000000
      }
    }
  },
  "default_topics": {
    "allowed": "confluent-audit-log-events",
    "denied": "confluent-audit-log-events"
  },
  "metadata": {
    "resource_version": "L38PPqakrbU5zX1bqdjz_A",
    "modified_since": "2024-09-29T13:44:55Z"
  }
}
```

Let's edit it and change to something like this:

```json
{
  "destinations": {
    "topics": {
      "confluent-audit-log-events": {
        "retention_ms": 7776000000
      }
    }
  },
  "default_topics": {
    "allowed": "confluent-audit-log-events",
    "denied": "confluent-audit-log-events"
  },
  "routes": {
    "crn:///kafka=*/topic=*": {
        "produce": {
            "allowed": "confluent-audit-log-events",
            "denied": "confluent-audit-log-events"
        },
        "consume": {
            "allowed": "confluent-audit-log-events",
            "denied": "confluent-audit-log-events"
        }
    }
  },
  "metadata": {
    "resource_version": "L38PPqakrbU5zX1bqdjz_A",
    "modified_since": "2024-09-29T13:44:55Z"
  }
}
```

Now we can update:

```shell
confluent audit-log config update < ./config.json
```

Confirm new logs for Read and Write operations show up:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events | jq '.time + "," + .data.authenticationInfo.principal + "," + .data.authorizationInfo.operation'
```

Download again locally:

```shell
confluent audit-log config describe > config.json
```

And edit to:

```json
{
  "destinations": {
    "topics": {
      "confluent-audit-log-events": {
        "retention_ms": 7776000000
      }
    }
  },
  "default_topics": {
    "allowed": "confluent-audit-log-events",
    "denied": "confluent-audit-log-events"
  },
  "routes": {
    "crn:///kafka=*/topic=*": {
      "authorize": {
        "allowed": null,
        "denied": null
      },
      "management": {
        "allowed": null,
        "denied": null
      },
      "produce": {
        "allowed": "",
        "denied": ""
      },
      "consume": {
        "allowed": "",
        "denied": ""
      },
      "describe": {
        "allowed": "",
        "denied": ""
      }
    }
  },
  "metadata": {
    "resource_version": "WDdDTDqv1ZJRIHNaNYKZyg",
    "updated_at": "2024-09-30T14:42:04Z"
  }
}
```

Update again:

```shell
confluent audit-log config update < ./config.json
```

In another shell execute:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_confkafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events | jq
```

We want to see the details of each audit. You will have Describe operations but check resourceType and it should be Cluster and not topic.

If you execute:

```shell
kafka-topics --bootstrap-server localhost:9092 --list --command-config ../delta_configs/client.properties.delta
```

No describe operation related to topics should appear.

Update the config to:

```json
{
  "destinations": {
    "topics": {
      "confluent-audit-log-events": {
        "retention_ms": 7776000000
      }
    }
  },
  "default_topics": {
    "allowed": "confluent-audit-log-events",
    "denied": "confluent-audit-log-events"
  },
  "routes": {
    "crn:///kafka=*/topic=*": {
      "authorize": {
        "allowed": null,
        "denied": null
      },
      "management": {
        "allowed": null,
        "denied": null
      },
      "produce": {
        "allowed": "",
        "denied": ""
      },
      "consume": {
        "allowed": "",
        "denied": ""
      },
      "describe": {
        "allowed": "confluent-audit-log-events",
        "denied": "confluent-audit-log-events"
      }
    }
  },
  "metadata": {
    "resource_version": "WDdDTDqv1ZJRIHNaNYKZyg",
    "updated_at": "2024-09-30T14:42:04Z"
  }
}
```

```shell
confluent audit-log config update < ./config.json
```

Execute again: 

```shell
kafka-topics --bootstrap-server localhost:9092 --list --command-config ../delta_configs/client.properties.delta
```

Now you should see entries related to DescribeConfigs and Describe with resourceType Topic.

## Cleanup

```shell
confluent logout
./cleanup.sh
cd ../../../..
rm -fr examples
```
