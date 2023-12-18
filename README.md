## Steps to create a RHDH dynamic frontend wrapper plugin

1. Clone this repository
2. Create `app-config.yaml` at the project root with the following contents:
```
app:
  baseUrl: "httpd://rh1plugin.com"
```
3. Execute the following command:
```
yarn install
yarn tsc
yarn build
yarn run export-dynamic
yarn publish 
```
4. Create a RHDH setup using Helm charts with the following configuration after replaceing the value of `clusterRouterBase` with the relevant value:
```
global:
  auth:
    backend:
      enabled: true
      existingSecret: ''
      value: ''
  clusterRouterBase: <REPLACE>
  dynamic:
    includes:
      - dynamic-plugins.default.yaml
    plugins:
      - integrity: >-
          sha512-v+R2v0N0bWhJ9LVXy8WcJpl7TfvvPOPZbJe1O8Ej+2+A63FUevk3q9kCi5G1NB8EmSg52hgsv1KgUg1ZR+I7fg==
        package: '@bbalasub/rh1-devquote-plugin@0.2.3'
        pluginConfig:
          dynamicPlugins:
            frontend:
              bbalasub.rh1-devquote-plugin:
                dynamicRoutes:
                  - importName: DevQuote
                    menuItem:
                      text: Quote
                    path: /devquote
                mountPoints:
                  - config:
                      layout:
                        gridColumnEnd:
                          lg: span 4
                          md: span 6
                          xs: span 12
                    importName: DevQuote
                    mountPoint: entity.page.overview/cards
  host: ''
route:
  annotations: {}
  enabled: true
  host: '{{ .Values.global.host }}'
  path: /
  tls:
    caCertificate: ''
    certificate: ''
    destinationCACertificate: ''
    enabled: true
    insecureEdgeTerminationPolicy: Redirect
    key: ''
    termination: edge
  wildcardPolicy: None
upstream:
  backstage:
    appConfig:
      app:
        baseUrl: 'https://{{- include "janus-idp.hostname" . }}'
      backend:
        auth:
          keys:
            - secret: '${BACKEND_SECRET}'
        baseUrl: 'https://{{- include "janus-idp.hostname" . }}'
        cors:
          origin: 'https://{{- include "janus-idp.hostname" . }}'
        database:
          connection:
            password: '${POSTGRESQL_ADMIN_PASSWORD}'
            user: postgres
      catalog:
        locations:
          - target: >-
              https://github.com/bbalakriz/rhdh-demo-catalog/blob/master/catalog-entities/all.yaml
            type: url
    args:
      - '--config'
      - dynamic-plugins-root/app-config.dynamic-plugins.yaml
    command: []
    extraEnvVars:
      - name: BACKEND_SECRET
        valueFrom:
          secretKeyRef:
            key: backend-secret
            name: '{{ include "janus-idp.backend-secret-name" $ }}'
      - name: POSTGRESQL_ADMIN_PASSWORD
        valueFrom:
          secretKeyRef:
            key: postgres-password
            name: '{{- include "janus-idp.postgresql.secretName" . }}'
    extraVolumeMounts:
      - mountPath: /opt/app-root/src/dynamic-plugins-root
        name: dynamic-plugins-root
    extraVolumes:
      - ephemeral:
          volumeClaimTemplate:
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 1Gi
        name: dynamic-plugins-root
      - configMap:
          defaultMode: 420
          name: dynamic-plugins
          optional: true
        name: dynamic-plugins
      - name: dynamic-plugins-npmrc
        secret:
          defaultMode: 420
          optional: true
          secretName: dynamic-plugins-npmrc
    image:
      pullSecrets: []
      registry: registry.redhat.io
      repository: rhdh/rhdh-hub-rhel9
      tag: 1.0-200
    initContainers:
      - command:
          - ./install-dynamic-plugins.sh
          - /dynamic-plugins-root
        env:
          - name: NPM_CONFIG_USERCONFIG
            value: /opt/app-root/src/.npmrc.dynamic-plugins
        image: '{{ include "backstage.image" . }}'
        imagePullPolicy: Always
        name: install-dynamic-plugins
        volumeMounts:
          - mountPath: /dynamic-plugins-root
            name: dynamic-plugins-root
          - mountPath: /opt/app-root/src/dynamic-plugins.yaml
            name: dynamic-plugins
            readOnly: true
            subPath: dynamic-plugins.yaml
          - mountPath: /opt/app-root/src/.npmrc.dynamic-plugins
            name: dynamic-plugins-npmrc
            readOnly: true
            subPath: .npmrc
        workingDir: /opt/app-root/src
    installDir: /opt/app-root/src
    livenessProbe:
      failureThreshold: 3
      httpGet:
        path: /healthcheck
        port: 7007
        scheme: HTTP
      initialDelaySeconds: 60
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 2
    podAnnotations:
      checksum/dynamic-plugins: >-
        {{- include "common.tplvalues.render" ( dict "value"
        .Values.global.dynamic "context" $) | sha256sum }}
    readinessProbe:
      failureThreshold: 3
      httpGet:
        path: /healthcheck
        port: 7007
        scheme: HTTP
      initialDelaySeconds: 30
      periodSeconds: 10
      successThreshold: 2
      timeoutSeconds: 2
  ingress:
    host: '{{ .Values.global.host }}'
  nameOverride: developer-hub
  postgresql:
    auth:
      secretKeys:
        adminPasswordKey: postgres-password
        userPasswordKey: password
    enabled: true
    image:
      registry: registry.redhat.io
      repository: rhel9/postgresql-15
      tag: latest
    postgresqlDataDir: /var/lib/pgsql/data/userdata
    primary:
      containerSecurityContext:
        enabled: false
      extraEnvVars:
        - name: POSTGRESQL_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              key: postgres-password
              name: '{{- include "postgresql.v1.secretName" . }}'
      persistence:
        enabled: true
        mountPath: /var/lib/pgsql/data
        size: 1Gi
      podSecurityContext:
        enabled: false
```
