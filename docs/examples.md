# Druid Operator Examples

## Example NodeSpec Supporting Multiple Key/Values

```yaml
    middlemanagers:
      podAnnotations:
        type: middlemanager
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-type
                    operator: In
                    values:
                      - druid-data
      tolerations:
         effect: NoSchedule
         key: node-role.kubernetes.io/master
         operator: Exists
      druid.port: 8091
      extra.jvm.options: |-
          -Xmx4G
          -Xms4G
      nodeType: middleManager
      nodeConfigMountPath: /opt/druid/conf/druid/cluster/data/middleManager
      podDisruptionBudgetSpec:
        maxUnavailable: 1
      ports:
        - containerPort: 8100
          name: peon-0
      replicas: 1
      resources:
        limits:
          cpu: "2"
          memory: 5Gi
        requests:
          cpu: "2"
          memory: 5Gi
      livenessProbe:
          initialDelaySeconds: 30
          httpGet:
            path: /status/health
            port: 8091
      readinessProbe:
          initialDelaySeconds: 30
          httpGet:
            path: /status/health
            port: 8091
      runtime.properties: |-
          druid.service=druid/middleManager
          druid.worker.capacity=4
          druid.indexer.task.baseTaskDir=/druid/data/baseTaskDir
          druid.server.http.numThreads=10
          druid.indexer.fork.property.druid.processing.buffer.sizeBytes=1
          druid.indexer.fork.property.druid.processing.numMergeBuffers=1
          druid.indexer.fork.property.druid.processing.numThreads=1
          # Processing threads and buffers on Peons
          druid.indexer.fork.property.druid.processing.numMergeBuffers=2
          druid.indexer.fork.property.druid.processing.buffer.sizeBytes=100000000
          druid.indexer.fork.property.druid.processing.numThreads=1
      services:
        - spec:
            clusterIP: None
            ports:
              - name: tcp-service-port
                port: 8091
                targetPort: 8091
            type: ClusterIP
      volumeClaimTemplates:
        - metadata:
            name: data-volume
          spec:
            accessModes:
              - ReadWriteOnce
            resources:
              requests:
                storage: 30Gi
            storageClassName: zone-a-storage
      volumeMounts:
        - mountPath: /druid/data
          name: data-volume
      containerSecurityContext:
        fsGroup: 1001
        capabilites:
          drop: ["ALL"]
      securityContext:
        runAsUser: 1001
      terminationGracePeriods: 100
      hpAutoscaler:
        maxReplicas: 10
        minReplicas: 1
        scaleTargetRef:
           apiVersion: apps/v1
           kind: StatefulSet
           name: druid-cluster-middlemanagers
        metrics:
         - type: Resource
           resource:
             name: cpu
             target:
               type: Utilization
               averageUtilization: 50
```

## Configure Ingress

```yaml
brokers:
  nodeType: "broker"
  druid.port: 8080
  ingressAnnotations:
      "nginx.ingress.kubernetes.io/rewrite-target": "/"
  ingress:
    ingressClassName: nginx # specific to ingress controllers.
    rules:
    - host: broker.myhostname.com
      http:
        paths:
        - backend:
            service:
              name: broker_svc
              port:
                name: http
          path: /
          pathType: ImplementationSpecific
    tls:
    - hosts:
      - broker.myhostname.com
      secretName: tls-broker-druid-cluster
```

## Configure Deployments

```yaml
  nodes:
    brokers:
      kind: Deployment
      nodeType: "broker"
      # Optionally specify for broker nodes
      # imagePullSecrets:
      # - name: tutu
      druid.port: 8088
      nodeConfigMountPath: "/opt/druid/conf/druid/cluster/query/broker"
      defaultReplicas: 1
```

## Configure Hot/Cold for Historicals

```yaml
    hot:
    druid.port: 8083
    env:
    - name: DRUID_XMS
        value: 2g
    - name: DRUID_XMX
        value: 2g
    - name: DRUID_MAXDIRECTMEMORYSIZE
        value: 2g
    livenessProbe:
        failureThreshold: 3
        httpGet:
        path: /status/health
        port: 8083
        initialDelaySeconds: 1800
        periodSeconds: 5
    nodeConfigMountPath: /opt/druid/conf/druid/cluster/data/historical
    nodeType: historical
    podDisruptionBudgetSpec:
        maxUnavailable: 1
    readinessProbe:
        failureThreshold: 18
        httpGet:
        path: /druid/historical/v1/readiness
        port: 8083
        periodSeconds: 10
    replicas: 1
    resources:
        limits:
        cpu: 3
        memory: 6Gi
        requests:
        cpu: 1
        memory: 1Gi
    runtime.properties:
        druid.plaintextPort=8083
        druid.service=druid/historical/hot
        ...
        ...
        ...

     cold:
        druid.port: 8083
        env:
        - name: DRUID_XMS
          value: 1500m
        - name: DRUID_XMX
          value: 1500m
        - name: DRUID_MAXDIRECTMEMORYSIZE
          value: 2g
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /status/health
            port: 8083
          initialDelaySeconds: 1800
          periodSeconds: 5
        nodeConfigMountPath: /opt/druid/conf/druid/cluster/data/historical
        nodeType: historical
        podDisruptionBudgetSpec:
          maxUnavailable: 1
        readinessProbe:
          failureThreshold: 18
          httpGet:
            path: /druid/historical/v1/readiness
            port: 8083
          periodSeconds: 10
        replicas: 1
        resources:
          limits:
            cpu: 4
            memory: 3.5Gi
          requests:
            cpu: 1
            memory: 1Gi
        runtime.properties:
          druid.plaintextPort=8083
          druid.service=druid/historical/cold
          ...
          ...
          ...
```

## Configure Additional Containers

```yaml
  additionalContainer:
    - image: universalforwarder-sidekick:next
      containerName: forwarder
      command:
        - /bin/sidekick
      imagePullPolicy: Always
      securityContext:
        runAsUser: 506
      volumeMounts:
        - name: logstore
          mountPath: /logstore
      env:
        - name: SAMPLE_ENV
          value: SAMPLE_VALUE
      resources:
        requests:
          memory: "1Gi"
          cpu: "500m"
        limits:
          memory: "1Gi"
          cpu: "500m"
      args:
        - -loggingEnabled=true
        - -dataCenter=dataCenter
        - -environment=environment
        - -application=application
        - -instance=instance
        - -logFiles=logFiles
```

## Using Environment Variables in common.runtime.properties

You can use environment variables in the `*.properties` section of your Druid configuration. This feature allows for more flexible and secure configuration management, especially for sensitive information or values that might change between environments.

To use an environment variable, use the following format:

```yaml
property_name={"type":"environment","variable":"ENVIRONMENT_VARIABLE_NAME"}
```

Here's an example of how you might use this in your `common.runtime.properties`:

```yaml
common.runtime.properties: |
  # Zookeeper
  druid.zk.service.host=tiny-cluster-zk-0.tiny-cluster-zk
  druid.zk.paths.base=/druid
  druid.zk.service.compress=false

  # Metadata Store
  druid.metadata.storage.type=derby
  druid.metadata.storage.connector.connectURI={"type":"environment","variable":"CONNECTOR_URI"}
  druid.metadata.storage.connector.host={"type":"environment","variable":"DB_HOST"}
  druid.metadata.storage.connector.port={"type":"environment","variable":"DB_PORT"}
  druid.metadata.storage.connector.createTables=true

```

In this example, values for `CONNECTOR_URI`, `DB_HOST`, `DB_PORT` will be pulled from environment variables, allowing you to set these values separately from your Druid configuration file.

Remember to ensure that these environment variables are properly set in your Kubernetes environment, typically through Secrets or ConfigMaps, before deploying your Druid cluster.

```yaml
  env:
    - name: CONNECTOR_URI
      valueFrom:
        secretKeyRef:
          key: connector-uri
          name: druid-secrets
```
