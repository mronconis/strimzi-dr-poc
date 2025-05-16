
# AMQ Streams




# AMQ Broker


1. [mirroring](https://docs.redhat.com/en/documentation/red_hat_amq_broker/7.12/html/deploying_amq_broker_on_openshift/assembly-br-configuring-operator-based-deployments_broker-ocp#proc-br-configuring-mirroring_broker-ocp)
2. [producing-and-consuming-test-messages](https://docs.redhat.com/en/documentation/red_hat_amq_broker/7.12/html/getting_started_with_amq_broker/creating-standalone-getting-started#producing-consuming-test-messages-getting-started)

## Test

Producer:
```bash
amq-broker/bin/artemis producer \
    --destination helloworld \
    --message-count 10 \
    --url tcp://amq-broker-amqp-acceptor-1-svc.amqbroker-cluster-live.svc.cluster.local:61617 \
    --protocol AMQP
```

Perf test:
```bash
amq-broker/bin/artemis perf client queue://helloworld \
    --url tcp://amq-broker-amqp-acceptor-1-svc.amqbroker-cluster-live.svc.cluster.local:61617 \
    --show-latency \
    --persistent \
    --warmup 5 \
    --message-size 1024 \
    --duration 95 \
    --rate 1000 \
    --commit-interval 1 \
    --enable-msg-id \
    --protocol AMQP \
    --max-pending 1 \
    --num-destinations 1 \
    --producers 1 \
    --threads 1 \
    --consumers 1 \
    --verbose
```

output:
```
--- SUMMARY
--- result:              success
--- total sent:            24981
--- total blocked:         24980
--- total completed:       24981
--- total received:      2330180 
--- aggregated delay send time: mean: 37325613.12 us - 50.00%: 38010879.00 us - 90.00%: 64487423.00 us - 99.00%: 70254591.00 us - 99.90%: 70778879.00 us - 99.99%: 70778879.00 us - max:    70778879.00 us 
--- aggregated send time:       mean:    485.42 us - 50.00%:    301.00 us - 90.00%:    619.00 us - 99.00%:   3231.00 us - 99.90%:  13759.00 us - 99.99%: 112127.00 us - max:    250879.00 us 
--- aggregated transfer time:   mean:   4986.61 us - 50.00%:   4415.00 us - 90.00%:   5951.00 us - 99.00%:  13631.00 us - 99.90%:  47103.00 us - 99.99%: 327679.00 us - max:    348159.00 us
```