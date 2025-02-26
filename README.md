# CP Active Users and Operations Auditing/Exclusion

- [CP Active Users and Operations Auditing/Exclusion](#cp-active-users-and-operations-auditingexclusion)
  - [Disclaimer](#disclaimer)
  - [Setup](#setup)
  - [Fetch Users per time and operation](#fetch-users-per-time-and-operation)
  - [Only the unique users from today](#only-the-unique-users-from-today)
  - [Audit Configuration](#audit-configuration)
    - [Exclude describe operations](#exclude-describe-operations)
      - [What if we log the rest of operations](#what-if-we-log-the-rest-of-operations)
  - [Cleanup](#cleanup)
  - [For a non MDS setup](#for-a-non-mds-setup)
    - [Capture Non Internal Topic Events (except describe)](#capture-non-internal-topic-events-except-describe)
    - [Cleanup](#cleanup-1)

## Disclaimer

The code and/or instructions here available are **NOT** intended for production usage. 
It's only meant to serve as an example or reference and does not replace the need to follow actual and official documentation of referenced products.

## Setup

Setup demo environment as per https://docs.confluent.io/platform/current/security/authorization/rbac/cp-rbac-example.html (please check requirements https://docs.confluent.io/platform/current/security/authorization/rbac/cp-rbac-example.html#prerequisites).

```shell
git clone https://github.com/confluentinc/examples.git
cd examples
git checkout 7.7.1-post
cd security/rbac/scripts
echo "Running Init Script..."
./init.sh
echo "Enabling RBAC..."
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

## Audit Configuration

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
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events | jq
```

We want to see the details of each audit. You will have Describe operations but check resourceType and it should be Cluster and not topic.

If you execute:

```shell
kafka-topics --bootstrap-server localhost:9092 --list --command-config ../delta_configs/client.properties.delta
```

No describe operation related to topics should appear.

Run again:

```shell
confluent audit-log config describe > config.json
```

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

### Exclude describe operations

Let's execute in a separate shell for monitoring specifics of audit events:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events | jq '.time + "," + .data.authenticationInfo.principal + "," + .data.authorizationInfo.operation + "," + .data.methodName + "," + .data.authorizationInfo.resourceType + "," + .data.authorizationInfo.resourceName'
```

Now if you execute:

```shell
kafka-consumer-groups --bootstrap-server localhost:9092 --list --command-config ../delta_configs/client.properties.delta 
```

You get past consumer groups. If you start in a separate shell:

```shell
kafka-console-consumer --topic topic1 --bootstrap-server localhost:9092 --consumer.config ../delta_configs/client.properties.delta --from-beginning --property print.key=true
```

We can then execute and see the new consumer group:

```shell
kafka-consumer-groups --bootstrap-server localhost:9092 --list --command-config ../delta_configs/client.properties.delta 
```

And pass that consumer group id into:

```shell
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --command-config ../delta_configs/client.properties.delta --group console-consumer-42110
```

We should see on our logs different audit events related to the topic1: Describe operation.

Let's execute:

```shell
confluent audit-log config describe > config.json
```

And update our audit configuration to:

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
    "resource_version": "B1LCQd_yo8BW7iHwCVWNGA",
    "updated_at": "2024-10-01T08:57:02Z"
  }
}
```

So executing:

```shell
confluent audit-log config update < ./config.json
```

Now if we execute again: 

```shell
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --command-config ../delta_configs/client.properties.delta --group console-consumer-42110
```

We should not see logged any Describe or DescribeConfigs related to topic1 while monitoring the audit log.

#### What if we log the rest of operations

Let's execute:

```shell
confluent audit-log config describe > config.json
```

And update our config to:

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
        "allowed": "confluent-audit-log-events",
        "denied": "confluent-audit-log-events"
      },
      "management": {
        "allowed": "confluent-audit-log-events",
        "denied": "confluent-audit-log-events"
      },
      "produce": {
        "allowed": "confluent-audit-log-events",
        "denied": "confluent-audit-log-events"
      },
      "consume": {
        "allowed": "confluent-audit-log-events",
        "denied": "confluent-audit-log-events"
      },
      "describe": {
        "allowed": "",
        "denied": ""
      }
    },
    "crn:///kafka=*/topic=confluent-audit-log-events": {
      "authorize": {
        "allowed": "",
        "denied": ""
      },
      "management": {
        "allowed": "",
        "denied": ""
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
    },
    "crn:///kafka=*/topic=_confluent*": {
      "authorize": {
        "allowed": "",
        "denied": ""
      },
      "management": {
        "allowed": "",
        "denied": ""
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
    "resource_version": "B1LCQd_yo8BW7iHwCVWNGA",
    "updated_at": "2024-10-01T08:57:02Z"
  }
}
```

(We are excluding logs from internal topics also as well from the audit log topic itself.)

Executing:

```shell
confluent audit-log config update < ./config.json
```

We are going to start having too many audit logs so we change our monitoring to:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config ../delta_configs/clientsma.properties.delta --from-beginning --topic confluent-audit-log-events | jq '.time + "," + .data.authenticationInfo.principal + "," + .data.authorizationInfo.operation + "," + .data.methodName + "," + .data.authorizationInfo.resourceType + "," + .data.authorizationInfo.resourceName' | grep 'Describe'
```

Then if we execute again: 

```shell
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --command-config ../delta_configs/client.properties.delta --group console-consumer-42110
```

If you wait a bit you should see a line on your monitoring corresponding to a describe operation on the topic1 related to `kafka.ListOffsets` as methodName:

```
"2024-11-10T12:52:12.260623Z,User:clienta,Describe,kafka.ListOffsets,Topic,topic1"
```

Which doesnt seem consistent with the fact that we have excluded the describe operation for our topics.

## Cleanup


Make sure you are at `examples/security/rbac/scripts`:

```shell
confluent logout
./cleanup.sh
cd ../../../..
rm -fr examples
```

## For a non MDS setup

Now we will check how to configure audit logging for non MDS setup.

We will be using our local `compose.yml`.

If you were using `kafka.security.authorizer.AclAuthorizer` as `KAFKA_AUTHORIZER_CLASS_NAME` you will need to change that to `io.confluent.kafka.security.authorizer.ConfluentServerAuthorizer`.

So we start CP:

```shell
docker compose up -d
```

And once started you should see in Control Center the topic `confluent-audit-log-events` with messages populated.

Now we can change the audit log configuration. First we open a shell into one of the broker containers:

```shell
docker compose exec kafka bash
```

Next we can list the configuration to confirm there are no dynamic configurations:

```shell
kafka-configs --bootstrap-server kafka:29093 --command-config=/etc/kafka/kafka-user.properties --entity-type brokers --entity-default --describe
```

Now we add our new configuration from the file `./kafka/new.properties`:

```shell
kafka-configs --bootstrap-server kafka:29093 --command-config=/etc/kafka/kafka-user.properties --entity-type brokers --entity-default --alter --add-config-file /etc/kafka/new.properties
```

You can list again the changes to confirm they were applied:

```shell
kafka-configs --bootstrap-server kafka:29093 --command-config=/etc/kafka/kafka-user.properties --entity-type brokers --entity-default --describe
```

You should see there wont be any more audit messages being produced to the audit log topic `confluent-audit-log-events`.

If we want we can revert by deleting our dynamic configuration:

```shell
kafka-configs --bootstrap-server kafka:29093 --command-config=/etc/kafka/kafka-user.properties --entity-type brokers --entity-default --alter --delete-config confluent.security.event.router.config,bootstrap.servers
```

The new audit log messages should start a bit after to show up on our topic.

We can exit now from our container shell.

### Capture Non Internal Topic Events (except describe)

Make sure to run:

```shell
kafka-configs --bootstrap-server localhost:9092 --command-config=./kafka/kafka-user.properties --entity-type brokers --entity-default --alter --add-config-file ./kafka/producer.properties
```

This will update the audit config for capturing all events execpt describe for non internal topics (as an example) using `kafka/producer.properties` (it also exclude events associated with the audit log topic itself).

Now if you produce through Control Center into a new topic named `test` or through command line:

```shell
kakafka-console-producer --broker-list localhost:9092 --topic test --producer.config ./kafka/kafka-user.properties 
```

After some seconds you should see the events captured while consuming from audit log topic:

```shell
kafka-console-consumer --bootstrap-server localhost:9092 --consumer.config kafka/kafka-user.properties --from-beginning --topic confluent-audit-log-events | jq '.time + "," + .data.authenticationInfo.principal + "," + .data.authorizationInfo.operation + ","+ .data.resourceName' | grep 'test'
```

### Cleanup

```shell
docker compose down -v
```