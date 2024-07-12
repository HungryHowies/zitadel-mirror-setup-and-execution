The following steps show my completed setup prior to executing the mirror command. My goal is to mirror data from  CockroachDB to PostgreSQL.
I was unable to mirror successfully and unsure where I am failing.

Using the following documantion for Mirror setup.

https://zitadel.com/docs/self-hosting/manage/cli/mirror#use-cases

### Environment:
```
Ubuntu 22.0.4
Zitadel version v2.54.6
CockroachDB CCL 23.2.7 (tried v24) && PostgreSQL 16
2 CPU and 4GB RAM
30 GB  drive
Nginx  local installation                                                                                                                             
Let’s encrypt  configured
```
### CockroachDB
```
curl https://binaries.cockroachdb.com/cockroach-v23.2.7.linux-amd64.tgz  | tar -zx && sudo cp -i cockroach-v23.2.7.linux-amd64/cockroach /usr/local/bin
```
```
sudo mkdir -p /usr/local/lib/cockroach
```
```  
sudo cp -i cockroach-v23.2.7.linux-amd64/lib/libgeos.so /usr/local/lib/cockroach/
```
```
sudo cp -i cockroach-v23.2.7.linux-amd64/lib/libgeos_c.so /usr/local/lib/cockroach/
```
```
which cockroach
```
```
cockroach start-single-node --insecure --background --http-addr :9000 --listen-addr=localhost
```
### Zitadel
```
wget https://github.com/zitadel/zitadel/releases/download/v2.54.6/zitadel-linux-amd64.tar.gz
```
```
tar -zxvf zitadel-linux-amd64.tar.gz
```
```
sudo mv zitadel-linux-amd64/zitadel /usr/local/bin/
```
```
cd /usr/local/bin/
```
```
vi defaults.yaml
```

### Let's encrypt
```
sudo apt install certbot python3-certbot-nginx
```
```
sudo certbot --nginx
```
```
vi  /etc/nginx/sites-available/zitadel
```
```
cd /etc/nginx/sites-enabled
```
```
rm default
```
```
ln -s /etc/nginx/sites-available/zitadel  /etc/nginx/sites-enabled/zitadel
```
```
nginx -t
```


### Zitadel Default Configuration file
```
cd /usr/local/bin/
```
```
vi defaults.yaml 
```
Add the following to defaults.yaml file
```
Log:
  Level: info # ZITADEL_LOG_LEVEL
  Formatter:
    Format: text # ZITADEL_LOG_FORMATTER_FORMAT

# Exposes metrics on /debug/metrics
Metrics:
  # Select type otel (OpenTelemetry) or none (disables collection and endpoint)
  Type: otel # ZITADEL_METRICS_TYPE

Tracing:
  # Choose one in "otel", "google", "log" and "none"
  # Depending on the type there are different configuration options
  # for type 'otel' is used for standard [open telemetry](https://opentelemetry.io)
  # Fraction: 1.0
  # Endpoint: 'otel.collector.endpoint'
  #
  # type 'log' or '' disables tracing
  #
  # for type 'google'
  # ProjectID: ''
  # Fraction: 1.0
  Type: none # ZITADEL_TRACING_TYPE
  Fraction: 1.0 # ZITADEL_TRACING_FRACTION
  # The endpoint of the otel collector endpoint
  Endpoint: "" #ZITADEL_TRACING_ENDPOINT

Telemetry:
  # As long as Enabled is true, ZITADEL tries to send usage data to the configured Telemetry.Endpoints.
  # Data is projected by ZITADEL even if Enabled is false.
  # This means that switching this to true makes ZITADEL try to send past data.
  Enabled: false # ZITADEL_TELEMETRY_ENABLED
  # Push telemetry data to all these endpoints at least once using an HTTP POST request.
  # If one endpoint returns an unsuccessful response code or times out,
  # ZITADEL retries to push the data point to all configured endpoints until it succeeds.
  # Configure delivery guarantees and intervals in the section Projections.Customizations.Telemetry
  # The endpoints can be reconfigured at runtime.
  # Ten redirects are followed.
  # If you change this configuration at runtime, remaining data that is not successfully delivered to the old endpoints is sent to the new endpoints.
  Endpoints:
    - https://httpbin.org/post
  # These headers are sent with every request to the configured endpoints.
  # Configure headers by environment variable using a JSON string with header values as arrays, like this:
  # ZITADEL_TELEMETRY_HEADERS='{"header1": ["value1"], "header2": ["value2", "value3"]}'
  Headers: # ZITADEL_TELEMETRY_HEADERS
  # single-value: "single-value"
  # multi-value:
  #   - "multi-value-1"
  #   - "multi-value-2"
  # The maximum number of data points that are queried before they are sent to the configured endpoints.
  Limit: 100 # ZITADEL_TELEMETRY_LIMIT

# Port ZITADEL will listen on
Port: 8080 # ZITADEL_PORT
# ExternalPort is the port on which end users access ZITADEL.
# It can differ from Port e.g. if a reverse proxy forwards the traffic to ZITADEL
# Read more about external access: https://zitadel.com/docs/self-hosting/manage/custom-domain
ExternalPort: 443 # ZITADEL_EXTERNALPORT
# ExternalPort is the domain on which end users access ZITADEL.
# Read more about external access: https://zitadel.com/docs/self-hosting/manage/custom-domain

ExternalDomain: zitadel.{domain}.com <--- my FQDN

# ExternalSecure specifies if ZITADEL is exposed externally using HTTPS or HTTP.
# Read more about external access: https://zitadel.com/docs/self-hosting/manage/custom-domain
ExternalSecure: true # ZITADEL_EXTERNALSECURE
TLS:
  # If enabled, ZITADEL will serve all traffic over TLS (HTTPS and gRPC)
  # you must then also provide a private key and certificate to be used for the connection
  # either directly or by a path to the corresponding file
  Enabled: true # ZITADEL_TLS_ENABLED
  
  KeyPath: /usr/local/bin/private.pem
  
  CertPath: /usr/local/bin/fullchain.pem
  
  Cert: # ZITADEL_TLS_CERT

# Header name of HTTP2 (incl. gRPC) calls from which the instance will be matched
HTTP2HostHeader: ":authority" # ZITADEL_HTTP2HOSTHEADER
# Header name of HTTP1 calls from which the instance will be matched
HTTP1HostHeader: "host" # ZITADEL_HTTP1HOSTHEADER

WebAuthNName: ZITADEL # ZITADEL_WEBAUTHNNAME

Database:
  # ZITADEL manages three database connection pools.
  # The *ConnRatio settings define the ratio of how many connections from
  # MaxOpenConns and MaxIdleConns are used to push events and spool projections.
  # Remaining connection are used for queries (search).
  # Values may not be negative and the sum of the ratios must always be less than 1.
  # For example this defaults define 40 MaxOpenConns overall.
  # - 40*0.2=8 connections are allocated to the event pusher;
  # - 40*0.2=8 connections are allocated to the projection spooler;
  # - 40-(8+8)=24 connections are remaining for queries;
  EventPushConnRatio: 0.2 # ZITADEL_DATABASE_COCKROACH_EVENTPUSHCONNRATIO
  ProjectionSpoolerConnRatio: 0.2 # ZITADEL_DATABASE_COCKROACH_PROJECTIONSPOOLERCONNRATIO
  # CockroachDB is the default database of ZITADEL
  cockroach:
    Host: localhost # ZITADEL_DATABASE_COCKROACH_HOST
    Port: 26257 # ZITADEL_DATABASE_COCKROACH_PORT
    Database: zitadel # ZITADEL_DATABASE_COCKROACH_DATABASE
    MaxOpenConns: 40 # ZITADEL_DATABASE_COCKROACH_MAXOPENCONNS
    MaxIdleConns: 20 # ZITADEL_DATABASE_COCKROACH_MAXIDLECONNS
    MaxConnLifetime: 30m # ZITADEL_DATABASE_COCKROACH_MAXCONNLIFETIME
    MaxConnIdleTime: 5m # ZITADEL_DATABASE_COCKROACH_MAXCONNIDLETIME
    Options: "" # ZITADEL_DATABASE_COCKROACH_OPTIONS
    User:
      Username: zitadel # ZITADEL_DATABASE_COCKROACH_USER_USERNAME
      Password: "" # ZITADEL_DATABASE_COCKROACH_USER_PASSWORD
      SSL:
        Mode: disable # ZITADEL_DATABASE_COCKROACH_USER_SSL_MODE
        RootCert: "" # ZITADEL_DATABASE_COCKROACH_USER_SSL_ROOTCERT
        Cert: "" # ZITADEL_DATABASE_COCKROACH_USER_SSL_CERT
        Key: "" # ZITADEL_DATABASE_COCKROACH_USER_SSL_KEY
    Admin:
      # By default, ExistingDatabase is not specified in the connection string
      # If the connection resolves to a database that is not existing in your system, configure an existing one here
      # It is used in zitadel init to connect to cockroach and create a dedicated database for ZITADEL.
      ExistingDatabase: # ZITADEL_DATABASE_COCKROACH_ADMIN_EXISTINGDATABASE
      Username: root # ZITADEL_DATABASE_COCKROACH_ADMIN_USERNAME
      Password: "" # ZITADEL_DATABASE_COCKROACH_ADMIN_PASSWORD
      SSL:
        Mode: disable # ZITADEL_DATABASE_COCKROACH_ADMIN_SSL_MODE
        RootCert: "" # ZITADEL_DATABASE_COCKROACH_ADMIN_SSL_ROOTCERT
        Cert: "" # ZITADEL_DATABASE_COCKROACH_ADMIN_SSL_CERT
        Key: "" # ZITADEL_DATABASE_COCKROACH_ADMIN_SSL_KEY
  # Postgres is used as soon as a value is set
  # The values describe the possible fields to set values
  postgres:
    Host: # ZITADEL_DATABASE_POSTGRES_HOST
    Port: # ZITADEL_DATABASE_POSTGRES_PORT
    Database: # ZITADEL_DATABASE_POSTGRES_DATABASE
    MaxOpenConns: # ZITADEL_DATABASE_POSTGRES_MAXOPENCONNS
    MaxIdleConns: # ZITADEL_DATABASE_POSTGRES_MAXIDLECONNS
    MaxConnLifetime: # ZITADEL_DATABASE_POSTGRES_MAXCONNLIFETIME
    MaxConnIdleTime: # ZITADEL_DATABASE_POSTGRES_MAXCONNIDLETIME
    Options: # ZITADEL_DATABASE_POSTGRES_OPTIONS
    User:
      Username: # ZITADEL_DATABASE_POSTGRES_USER_USERNAME
      Password: # ZITADEL_DATABASE_POSTGRES_USER_PASSWORD
      SSL:
        Mode: # ZITADEL_DATABASE_POSTGRES_USER_SSL_MODE
        RootCert: # ZITADEL_DATABASE_POSTGRES_USER_SSL_ROOTCERT
        Cert: # ZITADEL_DATABASE_POSTGRES_USER_SSL_CERT
        Key: # ZITADEL_DATABASE_POSTGRES_USER_SSL_KEY
    Admin:
      # The default ExistingDatabase is postgres
      # If your db system doesn't have a database named postgres, configure an existing database here
      # It is used in zitadel init to connect to postgres and create a dedicated database for ZITADEL.
      ExistingDatabase: # ZITADEL_DATABASE_POSTGRES_ADMIN_EXISTINGDATABASE
      Username: # ZITADEL_DATABASE_POSTGRES_ADMIN_USERNAME
      Password: # ZITADEL_DATABASE_POSTGRES_ADMIN_PASSWORD
      SSL:
        Mode: # ZITADEL_DATABASE_POSTGRES_ADMIN_SSL_MODE
        RootCert: # ZITADEL_DATABASE_POSTGRES_ADMIN_SSL_ROOTCERT
        Cert: # ZITADEL_DATABASE_POSTGRES_ADMIN_SSL_CERT
        Key: # ZITADEL_DATABASE_POSTGRES_ADMIN_SSL_KEY

Machine:
  # Cloud-hosted VMs need to specify their metadata endpoint so that the machine can be uniquely identified.
  Identification:
    # Use private IP to identify machines uniquely
    PrivateIp:
      Enabled: true # ZITADEL_MACHINE_IDENTIFICATION_PRIVATEIP_ENABLED
    # Use hostname to identify machines uniquely
    # You want the process to be identified uniquely, so this works well in k8s where each pod gets its own
    # unique hostname, but not as well in some other hosting environments.
    Hostname:
      Enabled: false # ZITADEL_MACHINE_IDENTIFICATION_HOSTNAME_ENABLED
    # Use a webhook response to identify machines uniquely
    # Google Cloud Configuration
    Webhook:
      Enabled: true # ZITADEL_MACHINE_IDENTIFICATION_WEBHOOK_ENABLED
      Url: "http://metadata.google.internal/computeMetadata/v1/instance/id" # ZITADEL_MACHINE_IDENTIFICATION_WEBHOOK_URL
      Headers:
        "Metadata-Flavor": "Google"
    #
    # AWS EC2 IMDSv1 Configuration: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html
    # Webhook:
    #   Url: "http://169.254.169.254/latest/meta-data/ami-id" # ZITADEL_MACHINE_IDENTIFICATION_WEBHOOK_URL
    #
    # AWS ECS v4 Configuration: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-metadata-endpoint-v4.html
    # Webhook:
    #   Url: "${ECS_CONTAINER_METADATA_URI_V4}" # ZITADEL_MACHINE_IDENTIFICATION_WEBHOOK_URL
    #   JPath: "$.DockerId" # ZITADEL_MACHINE_IDENTIFICATION_WEBHOOK_JPATH
    #
    # Azure Configuration: https://docs.microsoft.com/en-us/azure/virtual-machines/windows/instance-metadata-service?tabs=linux
    # Webhook:
    #   Url: "http://169.254.169.254/metadata/instance?api-version=2021-02-01" # ZITADEL_MACHINE_IDENTIFICATION_WEBHOOK_URL
    #   JPath: "$.compute.vmId" # ZITADEL_MACHINE_IDENTIFICATION_WEBHOOK_JPATH

# Storage for assets like user avatar, organization logo, icon, font, ...
AssetStorage:
  Type: db # ZITADEL_ASSET_STORAGE_TYPE
  # HTTP cache control settings for serving assets in the assets API and login UI
  # the assets will also be served with an etag and last-modified header
  Cache:
    MaxAge: 5s # ZITADEL_ASSETSTORAGE_CACHE_MAXAGE
    # 168h are 7 days
    SharedMaxAge: 168h # ZITADEL_ASSETSTORAGE_CACHE_SHAREDMAXAGE

# The Projections section defines the behavior for the scheduled and synchronous events projections.
Projections:
  # The maximum duration a transaction remains open
  # before it spots left folding additional events
  # and updates the table.
  TransactionDuration: 500ms # ZITADEL_PROJECTIONS_TRANSACTIONDURATION
  # Time interval between scheduled projections
  RequeueEvery: 60s # ZITADEL_PROJECTIONS_REQUEUEEVERY
  # Time between retried database statements resulting from projected events
  RetryFailedAfter: 1s # ZITADEL_PROJECTIONS_RETRYFAILEDAFTER
  # Retried execution number of database statements resulting from projected events
  MaxFailureCount: 5 # ZITADEL_PROJECTIONS_MAXFAILURECOUNT
  # Limit of returned events per query
  BulkLimit: 200 # ZITADEL_PROJECTIONS_BULKLIMIT
  # Only instances are projected, for which at least a projection-relevant event exists within the timeframe
  # from HandleActiveInstances duration in the past until the projection's current time
  # If set to 0 (default), every instance is always considered active
  HandleActiveInstances: 0s # ZITADEL_PROJECTIONS_HANDLEACTIVEINSTANCES
  # In the Customizations section, all settings from above can be overwritten for each specific projection
  Customizations:
    Projects:
      TransactionDuration: 2s
    custom_texts:
      TransactionDuration: 2s
      BulkLimit: 400
    # The Notifications projection is used for sending emails and SMS to users
    Notifications:
      # As notification projections don't result in database statements, retries don't have an effect
      MaxFailureCount: 10 # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_NOTIFICATIONS_MAXFAILURECOUNT
      # Sending emails can take longer than 500ms
      TransactionDuration: 5s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_NOTIFICATIONS_TRANSACTIONDURATION
    password_complexities:
      TransactionDuration: 2s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_PASSWORD_COMPLEXITIES_TRANSACTIONDURATION
    lockout_policy:
      TransactionDuration: 2s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_LOCKOUT_POLICY_TRANSACTIONDURATION
    # The NotificationsQuotas projection is used for calling quota webhooks
    NotificationsQuotas:
      # In case of failed deliveries, ZITADEL retries to send the data points to the configured endpoints, but only for active instances.
      # An instance is active, as long as there are projected events on the instance, that are not older than the HandleActiveInstances duration.
      # Delivery guarantee requirements are higher for quota webhooks
      # If set to 0 (default), every instance is always considered active
      HandleActiveInstances: 0s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_NOTIFICATIONSQUOTAS_HANDLEACTIVEINSTANCES
      # As quota notification projections don't result in database statements, retries don't have an effect
      MaxFailureCount: 10 # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_NOTIFICATIONSQUOTAS_MAXFAILURECOUNT
      # Quota notifications are not so time critical. Setting RequeueEvery every five minutes doesn't annoy the db too much.
      RequeueEvery: 300s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_NOTIFICATIONSQUOTAS_REQUEUEEVERY
      # Sending emails can take longer than 500ms
      TransactionDuration: 5s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_NOTIFICATIONQUOTAS_TRANSACTIONDURATION
    milestones:
      BulkLimit: 50
    # The Telemetry projection is used for calling telemetry webhooks
    Telemetry:
      # In case of failed deliveries, ZITADEL retries to send the data points to the configured endpoints, but only for active instances.
      # An instance is active, as long as there are projected events on the instance, that are not older than the HandleActiveInstances duration.
      # Telemetry delivery guarantee requirements are a bit higher than normal data projections, as they are not interactively retryable.
      # If set to 0 (default), every instance is always considered active
      HandleActiveInstances: 0s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_TELEMETRY_HANDLEACTIVEINSTANCES
      # As sending telemetry data doesn't result in database statements, retries don't have any effects
      MaxFailureCount: 0 # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_TELEMETRY_MAXFAILURECOUNT
      # Telemetry data synchronization is not time critical. Setting RequeueEvery to 55 minutes doesn't annoy the database too much.
      RequeueEvery: 3300s # ZITADEL_PROJECTIONS_CUSTOMIZATIONS_TELEMETRY_REQUEUEEVERY

Auth:
  # See Projections.BulkLimit
  SearchLimit: 1000 # ZITADEL_AUTH_SEARCHLIMIT
  Spooler:
    # See Projections.TransationDuration
    TransactionDuration: 10s #ZITADEL_AUTH_SPOOLER_TRANSACTIONDURATION
    # See Projections.BulkLimit
    BulkLimit: 100 #ZITADEL_AUTH_SPOOLER_BULKLIMIT
    # See Projections.MaxFailureCount
    FailureCountUntilSkip: 5 #ZITADEL_AUTH_SPOOLER_FAILURECOUNTUNTILSKIP
    # Only instance are projected, for which at least a projection relevant event exists withing the timeframe
    # from HandleActiveInstances duration in the past until the projections current time
    # If set to 0 (default), every instance is always considered active
    HandleActiveInstances: 0s #ZITADEL_AUTH_SPOOLER_HANDLEACTIVEINSTANCES
  # Defines the amount of auth requests stored in the LRU caches.
  # There are two caches implemented one for id and one for code
  AmountOfCachedAuthRequests: 0 #ZITADEL_AUTH_AMOUNTOFCACHEDAUTHREQUESTS

Admin:
  # See Projections.BulkLimit
  SearchLimit: 1000 # ZITADEL_ADMIN_SEARCHLIMIT
  Spooler:
    # See Projections.TransationDuration
    TransactionDuration: 10s
    # See Projections.BulkLimit
    BulkLimit: 200
    # See Projections.MaxFailureCount
    FailureCountUntilSkip: 5
    # Only instance are projected, for which at least a projection relevant event exists withing the timeframe
    # from HandleActiveInstances duration in the past until the projections current time
    # If set to 0 (default), every instance is always considered active
    HandleActiveInstances: 0s

UserAgentCookie:
  Name: zitadel.useragent # ZITADEL_USERAGENTCOOKIE_NAME
  # 8760h are 365 days, one year
  MaxAge: 8760h # ZITADEL_USERAGENTCOOKIE_MAXAGE

OIDC:
  CodeMethodS256: true # ZITADEL_OIDC_CODEMETHODS256
  AuthMethodPost: true # ZITADEL_OIDC_AUTHMETHODPOST
  AuthMethodPrivateKeyJWT: true # ZITADEL_OIDC_AUTHMETHODPRIVATEKEYJWT
  GrantTypeRefreshToken: true # ZITADEL_OIDC_GRANTTYPEREFRESHTOKEN
  RequestObjectSupported: true # ZITADEL_OIDC_REQUESTOBJECTSUPPORTED
  SigningKeyAlgorithm: RS256 # ZITADEL_OIDC_SIGNINGKEYALGORITHM
  # Sets the default values for lifetime and expiration for OIDC
  # This default can be overwritten in the default instance configuration and for each instance during runtime
  # !!! Changing this after the initial setup will have no impact without a restart !!!
  DefaultAccessTokenLifetime: 12h # ZITADEL_OIDC_DEFAULTACCESSTOKENLIFETIME
  DefaultIdTokenLifetime: 12h # ZITADEL_OIDC_DEFAULTIDTOKENLIFETIME
  # 720h are 30 days, one month
  DefaultRefreshTokenIdleExpiration: 720h # ZITADEL_OIDC_DEFAULTREFRESHTOKENIDLEEXPIRATION
  # 2160h are 90 days, three months
  DefaultRefreshTokenExpiration: 2160h # ZITADEL_OIDC_DEFAULTREFRESHTOKENEXPIRATION
  Cache:
    MaxAge: 12h # ZITADEL_OIDC_CACHE_MAXAGE
    # 168h is 7 days, one week
    SharedMaxAge: 168h # ZITADEL_OIDC_CACHE_SHAREDMAXAGE
  CustomEndpoints:
    Auth:
      Path: /oauth/v2/authorize # ZITADEL_OIDC_CUSTOMENDPOINTS_AUTH_PATH
    Token:
      Path: /oauth/v2/token # ZITADEL_OIDC_CUSTOMENDPOINTS_TOKEN_PATH
    Introspection:
      Path: /oauth/v2/introspect # ZITADEL_OIDC_CUSTOMENDPOINTS_INTROSPECTION_PATH
    Userinfo:
      Path: /oidc/v1/userinfo # ZITADEL_OIDC_CUSTOMENDPOINTS_USERINFO_PATH
    Revocation:
      Path: /oauth/v2/revoke # ZITADEL_OIDC_CUSTOMENDPOINTS_REVOCATION_PATH
    EndSession:
      Path: /oidc/v1/end_session # ZITADEL_OIDC_CUSTOMENDPOINTS_ENDSESSION_PATH
    Keys:
      Path: /oauth/v2/keys # ZITADEL_OIDC_CUSTOMENDPOINTS_KEYS_PATH
    DeviceAuth:
      Path: /oauth/v2/device_authorization # ZITADEL_OIDC_CUSTOMENDPOINTS_DEVICEAUTH_PATH
  DefaultLoginURLV2: "/login?authRequest=" # ZITADEL_OIDC_DEFAULTLOGINURLV2
  DefaultLogoutURLV2: "/logout?post_logout_redirect=" # ZITADEL_OIDC_DEFAULTLOGOUTURLV2
  PublicKeyCacheMaxAge: 24h # ZITADEL_OIDC_PUBLICKEYCACHEMAXAGE

SAML:
  ProviderConfig:
    MetadataConfig:
      Path: "/metadata" # ZITADEL_SAML_PROVIDERCONFIG_METADATACONFIG_PATH
      SignatureAlgorithm: "http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" # ZITADEL_SAML_PROVIDERCONFIG_METADATACONFIG_SIGNATUREALGORITHM
    IDPConfig:
      SignatureAlgorithm: "http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" # ZITADEL_SAML_PROVIDERCONFIG_IDPCONFIG_SIGNATUREALGORITHM
      WantAuthRequestsSigned: true # ZITADEL_SAML_PROVIDERCONFIG_IDPCONFIG_WANTAUTHREQUESTSSIGNED
      Endpoints:
    #Organisation:
    #  Name: ZITADEL # ZITADEL_SAML_PROVIDERCONFIG_ORGANISATION_NAME
    #  URL: https://zitadel.com # ZITADEL_SAML_PROVIDERCONFIG_ORGANISATION_URL
    #ContactPerson:
    #  ContactType: "technical" # ZITADEL_SAML_PROVIDERCONFIG_CONTACTPERSON_CONTACTTYPE
    #  Company: ZITADEL # ZITADEL_SAML_PROVIDERCONFIG_CONTACTPERSON_COMPANY
    #  EmailAddress: hi@zitadel.com # ZITADEL_SAML_PROVIDERCONFIG_CONTACTPERSON_EMAILADDRESS

Login:
  LanguageCookieName: zitadel.login.lang # ZITADEL_LOGIN_LANGUAGECOOKIENAME
  CSRFCookieName: zitadel.login.csrf # ZITADEL_LOGIN_CSRFCOOKIENAME
  Cache:
    MaxAge: 12h # ZITADEL_LOGIN_CACHE_MAXAGE
    # 168h is 7 days, one week
    SharedMaxAge: 168h # ZITADEL_LOGIN_CACHE_SHAREDMAXAGE
  DefaultOTPEmailURLV2: "/otp/verify?loginName={{.LoginName}}&code={{.Code}}" # ZITADEL_LOGIN_CACHE_DEFAULTOTPEMAILURLV2

Console:
  ShortCache:
    MaxAge: 0m # ZITADEL_CONSOLE_SHORTCACHE_MAXAGE
    SharedMaxAge: 5m # ZITADEL_CONSOLE_SHORTCACHE_SHAREDMAXAGE
  LongCache:
    MaxAge: 12h # ZITADEL_CONSOLE_LONGCACHE_MAXAGE
    # 168h is 7 days, one week
    SharedMaxAge: 168h # ZITADEL_CONSOLE_LONGCACHE_SHAREDMAXAGE
  InstanceManagementURL: "" # ZITADEL_CONSOLE_INSTANCEMANAGEMENTURL

EncryptionKeys:
  DomainVerification:
    EncryptionKeyID: "domainVerificationKey" # ZITADEL_ENCRYPTIONKEYS_DOMAINVERIFICATION_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_DOMAINVERIFICATION_DECRYPTIONKEYIDS (comma separated list)
  IDPConfig:
    EncryptionKeyID: "idpConfigKey" # ZITADEL_ENCRYPTIONKEYS_IDPCONFIG_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_IDPCONFIG_DECRYPTIONKEYIDS (comma separated list)
  OIDC:
    EncryptionKeyID: "oidcKey" # ZITADEL_ENCRYPTIONKEYS_OIDC_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_OIDC_DECRYPTIONKEYIDS (comma separated list)
  SAML:
    EncryptionKeyID: "samlKey" # ZITADEL_ENCRYPTIONKEYS_SAML_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_SAML_DECRYPTIONKEYIDS (comma separated list)
  OTP:
    EncryptionKeyID: "otpKey" # ZITADEL_ENCRYPTIONKEYS_OTP_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_OTP_DECRYPTIONKEYIDS (comma separated list)
  SMS:
    EncryptionKeyID: "smsKey" # ZITADEL_ENCRYPTIONKEYS_SMS_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_SMS_DECRYPTIONKEYIDS (comma separated list)
  SMTP:
    EncryptionKeyID: "smtpKey" # ZITADEL_ENCRYPTIONKEYS_SMTP_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_SMTP_DECRYPTIONKEYIDS (comma separated list)
  User:
    EncryptionKeyID: "userKey" # ZITADEL_ENCRYPTIONKEYS_USER_ENCRYPTIONKEYID
    DecryptionKeyIDs: # ZITADEL_ENCRYPTIONKEYS_USER_DECRYPTIONKEYIDS (comma separated list)
  CSRFCookieKeyID: "csrfCookieKey" # ZITADEL_ENCRYPTIONKEYS_CSRFCOOKIEKEYID
  UserAgentCookieKeyID: "userAgentCookieKey" # ZITADEL_ENCRYPTIONKEYS_USERAGENTCOOKIEKEYID

SystemAPIUsers:
# # Add keys for authentication of the systemAPI here:
# # you can specify any name for the user, but they will have to match the `issuer` and `sub` claim in the JWT:
# - superuser:
#     Path: /path/to/superuser/ey.pem  # you can provide the key either by reference with the path
#     Memberships:
#       # MemberType System allows the user to access all APIs for all instances or organizations
#       - MemberType: System
#         Roles:
#           - "SYSTEM_OWNER"
#           # Actually, we don't recommend adding IAM_OWNER and ORG_OWNER to the System membership, as this basically enables god mode for the system user
#           - "IAM_OWNER"
#           - "ORG_OWNER"
#       # MemberType IAM and Organization let you restrict access to a specific instance or organization by specifying the AggregateID
#       - MemberType: IAM
#         Roles: "IAM_OWNER"
#         AggregateID: "123456789012345678"
#       - MemberType: Organization
#         Roles: "ORG_OWNER"
#         AggregateID: "123456789012345678"
# - superuser2:
#     # If no memberships are specified, the user has a membership of type System with the role "SYSTEM_OWNER"
#     KeyData: <base64 encoded key>     # or you can directly embed it as base64 encoded value
# Configure the SystemAPIUsers by environment variable using JSON notation:
# ZITADEL_SYSTEMAPIUSERS='{"systemuser":{"Path":"/path/to/superuser/key.pem"},"systemuser2":{"KeyData":"<base64 encoded key>"}}'

SystemDefaults:
  SecretGenerators:
    MachineKeySize: 2048 # ZITADEL_SYSTEMDEFAULTS_SECRETGENERATORS_MACHINEKEYSIZE
    ApplicationKeySize: 2048 # ZITADEL_SYSTEMDEFAULTS_SECRETGENERATORS_APPLICATIONKEYSIZE
  PasswordHasher:
    # Set hasher configuration for user passwords.
    # Passwords previously hashed with a different algorithm
    # or cost are automatically re-hashed using this config,
    # upon password validation or update.
    # Configure the Hasher config by environment variable using JSON notation:
    # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_HASHER='{"Algorithm":"pbkdf2","Rounds":290000,"Hash":"sha256"}'
    Hasher:
      # Supported algorithms: "argon2i", "argon2id", "bcrypt", "scrypt", "pbkdf2"
      # Depending on the algorithm, different configuration options take effect.
      Algorithm: bcrypt
      # Cost takes effect for the algorithms bcrypt and scrypt
      Cost: 14 # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_HASHER_COST
      # Time takes effect for the algorithms argon2i and argon2id
      Time: 3 # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_HASHER_TIME
      # Memory takes effect for the algorithms argon2i and argon2id
      Memory: 32768 # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_HASHER_MEMORY
      # Threads takes effect for the algorithms argon2i and argon2id
      Threads: 4 # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_HASHER_THREADS
      # Rounds takes effect for the algorithm pbkdf2
      Rounds: 290000 # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_HASHER_ROUNDS
      # Hash takes effect for the algorithm pbkdf2
      # Can be "sha1", "sha224", "sha256", "sha384" or "sha512"
      Hash: sha256 # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_HASHER_HASH

    # Verifiers enable the possibility of verifying
    # passwords that are previously hashed using another
    # algorithm then the Hasher.
    # This can be used when migrating from one algorithm to another,
    # or when importing users with hashed passwords.
    # There is no need to enable a Verifier of the same algorithm
    # as the Hasher.
    #
    # The format of the encoded hash strings must comply
    # with https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md
    # https://passlib.readthedocs.io/en/stable/modular_crypt_format.html
    #
    # Supported verifiers: (uncomment to enable)
    Verifiers: # ZITADEL_SYSTEMDEFAULTS_PASSWORDHASHER_VERIFIERS
    #   - "argon2"   # verifier for both argon2i and argon2id.
    #   - "bcrypt"
    #   - "md5"      # md5Crypt with salt and password shuffling.
    #   - "md5plain" # md5 digest of a password without salt
    #   - "scrypt"
    #   - "pbkdf2"   # verifier for all pbkdf2 hash modes.
  SecretHasher:
    # Set hasher configuration for machine users, API and OIDC client secrets.
    Hasher:
      # Supported algorithms: "argon2i", "argon2id", "bcrypt", "scrypt", "pbkdf2"
      # Depending on the algorithm, different configuration options take effect.
      Algorithm: bcrypt
      # Cost takes effect for the algorithms bcrypt and scrypt
      Cost: 4 # ZITADEL_SYSTEMDEFAULTS_SECRETHASHER_HASHER_COST
      # Time takes effect for the algorithms argon2i and argon2id
      Time: 3 # ZITADEL_SYSTEMDEFAULTS_SECRETHASHER_HASHER_TIME
      # Memory takes effect for the algorithms argon2i and argon2id
      Memory: 32768 # ZITADEL_SYSTEMDEFAULTS_SECRETHASHER_HASHER_MEMORY
      # Threads takes effect for the algorithms argon2i and argon2id
      Threads: 4 # ZITADEL_SYSTEMDEFAULTS_SECRETHASHER_HASHER_THREADS
      # Rounds takes effect for the algorithm pbkdf2
      Rounds: 290000 # ZITADEL_SYSTEMDEFAULTS_SECRETHASHER_HASHER_ROUNDS
      # Hash takes effect for the algorithm pbkdf2
      # Can be "sha1", "sha224", "sha256", "sha384" or "sha512"
      Hash: sha256 # ZITADEL_SYSTEMDEFAULTS_SECRETHASHER_HASHER_HASH
    Verifiers: # ZITADEL_SYSTEMDEFAULTS_SECRETHASHER_VERIFIERS
  Multifactors:
    OTP:
      # If this is empty, the issuer is the requested domain
      # This is helpful in scenarios with multiple ZITADEL environments or virtual instances
      Issuer: "ZITADEL" # ZITADEL_SYSTEMDEFAULTS_MULTIFACTORS_OTP_ISSUER
  DomainVerification:
    VerificationGenerator:
      Length: 32 # ZITADEL_SYSTEMDEFAULTS_DOMAINVERIFICATION_VERIFICATIONGENERATOR_LENGTH
      IncludeLowerLetters: true # ZITADEL_SYSTEMDEFAULTS_DOMAINVERIFICATION_VERIFICATIONGENERATOR_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_SYSTEMDEFAULTS_DOMAINVERIFICATION_VERIFICATIONGENERATOR_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_SYSTEMDEFAULTS_DOMAINVERIFICATION_VERIFICATIONGENERATOR_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_SYSTEMDEFAULTS_DOMAINVERIFICATION_VERIFICATIONGENERATOR_INCLUDESYMBOLS
  Notifications:
    FileSystemPath: ".notifications/" # ZITADEL_SYSTEMDEFAULTS_NOTIFICATIONS_FILESYSTEMPATH
  KeyConfig:
    Size: 2048 # ZITADEL_SYSTEMDEFAULTS_KEYCONFIG_SIZE
    CertificateSize: 4096 # ZITADEL_SYSTEMDEFAULTS_KEYCONFIG_CERTIFICATESIZE
    PrivateKeyLifetime: 6h # ZITADEL_SYSTEMDEFAULTS_KEYCONFIG_PRIVATEKEYLIFETIME
    PublicKeyLifetime: 30h # ZITADEL_SYSTEMDEFAULTS_KEYCONFIG_PUBLICKEYLIFETIME
    # 8766h are 1 year
    CertificateLifetime: 8766h # ZITADEL_SYSTEMDEFAULTS_KEYCONFIG_CERTIFICATELIFETIME

Actions:
  HTTP:
    # Wildcard sub domains are currently unsupported
    DenyList: # ZITADEL_ACTIONS_HTTP_DENYLIST (comma separated list)
      - localhost
      - "127.0.0.1"

LogStore:
  Access:
    Stdout:
      # If enabled, all access logs are printed to the binary's standard output
      Enabled: false # ZITADEL_LOGSTORE_ACCESS_STDOUT_ENABLED
  Execution:
    Stdout:
      # If enabled, all execution logs are printed to the binary's standard output
      Enabled: true # ZITADEL_LOGSTORE_EXECUTION_STDOUT_ENABLED

Quotas:
  Access:
    # If enabled, authenticated requests are counted and potentially limited depending on the configured quota of the instance
    Enabled: false # ZITADEL_QUOTAS_ACCESS_ENABLED
    Debounce:
      MinFrequency: 0s # ZITADEL_QUOTAS_ACCESS_DEBOUNCE_MINFREQUENCY
      MaxBulkSize: 0 # ZITADEL_QUOTAS_ACCESS_DEBOUNCE_MAXBULKSIZE
    ExhaustedCookieKey: "zitadel.quota.exhausted" # ZITADEL_QUOTAS_ACCESS_EXHAUSTEDCOOKIEKEY
    ExhaustedCookieMaxAge: "300s" # ZITADEL_QUOTAS_ACCESS_EXHAUSTEDCOOKIEMAXAGE
  Execution:
    # If enabled, all action executions are counted and potentially limited depending on the configured quota of the instance
    Enabled: false # ZITADEL_QUOTAS_EXECUTION_DATABASE_ENABLED
    Debounce:
      MinFrequency: 0s # ZITADEL_QUOTAS_EXECUTION_DEBOUNCE_MINFREQUENCY
      MaxBulkSize: 0 # ZITADEL_QUOTAS_EXECUTION_DEBOUNCE_MAXBULKSIZE

Eventstore:
  # Sets the maximum duration of transactions pushing events
  PushTimeout: 15s #ZITADEL_EVENTSTORE_PUSHTIMEOUT
  # Maximum amount of push retries in case of primary key violation on the sequence
  MaxRetries: 5 #ZITADEL_EVENTSTORE_MAXRETRIES

# The DefaultInstance section defines the default values for each new virtual instance that is created.
# Check out https://zitadel.com/docs/concepts/structure/instance#multiple-virtual-instances for more information about virtual instances.
# For the initial setup, the default values are used to create the first instance.
# However, you might want to have your first instance created by the setup job to have a different configuration.
# To overwrite the default values for the initial setup, configure the FirstInstance yaml section and pass it using the --steps flag.
DefaultInstance:
  InstanceName: ZITADEL # ZITADEL_DEFAULTINSTANCE_INSTANCENAME
  DefaultLanguage: en # ZITADEL_DEFAULTINSTANCE_DEFAULTLANGUAGE
  Org:
    Name: ZITADEL # ZITADEL_DEFAULTINSTANCE_ORG_NAME
    # In the DefaultInstance.Org.Human section, the initial organization's admin user with the role IAM_OWNER is defined.
    # If DefaultInstance.Org.Machine.Machine is defined, a service user is created with the IAM_OWNER role.
    Human:
      # In case that UserLoginMustBeDomain is false (default) and if you don't overwrite the username with an email,
      # it will be suffixed by the org domain (org-name + domain from config).
      # for example zitadel-admin in org `My Org` on domain.tld -> zitadel-admin@my-org.domain.tld
      UserName: zitadel-admin # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_USERNAME
      FirstName: ZITADEL # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_FIRSTNAME
      LastName: Admin # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_LASTNAME
      NickName: # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_NICKNAME
      DisplayName: # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_DISPLAYNAME
      Email:
        Address: # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_EMAIL_ADDRESS
        Verified: false # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_EMAIL_VERIFIED
      PreferredLanguage: en # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_PREFERREDLANGUAGE
      Gender: # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_GENDER
      Phone:
        Number: # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_PHONE_NUMBER
        Verified: # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_PHONE_VERIFIED
      Password: # ZITADEL_DEFAULTINSTANCE_ORG_HUMAN_PASSWORD
    # In the DefaultInstance.Org.Machine section, the initial organization's admin user with the role IAM_OWNER is defined.
    # If DefaultInstance.Org.Machine.Machine is defined, a service user is created with the IAM_OWNER role.
    Machine:
      Machine:
        Username: # ZITADEL_DEFAULTINSTANCE_ORG_MACHINE_MACHINE_USERNAME
        Name: # ZITADEL_DEFAULTINSTANCE_ORG_MACHINE_MACHINE_NAME
      MachineKey:
        # date format: 2023-01-01T00:00:00Z
        ExpirationDate: # ZITADEL_DEFAULTINSTANCE_ORG_MACHINE_MACHINEKEY_EXPIRATIONDATE
        # Currently, the only supported value is 1 for JSON
        Type: # ZITADEL_DEFAULTINSTANCE_ORG_MACHINE_MACHINEKEY_TYPE
      Pat:
        # date format: 2023-01-01T00:00:00Z
        ExpirationDate: # ZITADEL_DEFAULTINSTANCE_ORG_MACHINE_PAT_EXPIRATIONDATE
  SecretGenerators:
    ClientSecret:
      Length: 64 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_CLIENTSECRET_LENGTH
      IncludeLowerLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_CLIENTSECRET_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_CLIENTSECRET_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_CLIENTSECRET_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_CLIENTSECRET_INCLUDESYMBOLS
    InitializeUserCode:
      Length: 6 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_INITIALIZEUSERCODE_LENGTH
      Expiry: "72h" # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_INITIALIZEUSERCODE_EXPIRY
      IncludeLowerLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_INITIALIZEUSERCODE_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_INITIALIZEUSERCODE_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_INITIALIZEUSERCODE_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_INITIALIZEUSERCODE_INCLUDESYMBOLS
    EmailVerificationCode:
      Length: 6 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_EMAILVERIFICATIONCODE_LENGTH
      Expiry: "1h" # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_EMAILVERIFICATIONCODE_EXPIRY
      IncludeLowerLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_EMAILVERIFICATIONCODE_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_EMAILVERIFICATIONCODE_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_EMAILVERIFICATIONCODE_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_EMAILVERIFICATIONCODE_INCLUDESYMBOLS
    PhoneVerificationCode:
      Length: 6 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PHONEVERIFICATIONCODE_LENGTH
      Expiry: "1h" # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PHONEVERIFICATIONCODE_EXPIRY
      IncludeLowerLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PHONEVERIFICATIONCODE_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PHONEVERIFICATIONCODE_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PHONEVERIFICATIONCODE_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PHONEVERIFICATIONCODE_INCLUDESYMBOLS
    PasswordVerificationCode:
      Length: 6 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDVERIFICATIONCODE_LENGTH
      Expiry: "1h" # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDVERIFICATIONCODE_EXPIRY
      IncludeLowerLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDVERIFICATIONCODE_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDVERIFICATIONCODE_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDVERIFICATIONCODE_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDVERIFICATIONCODE_INCLUDESYMBOLS
    PasswordlessInitCode:
      Length: 12 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDLESSINITCODE_LENGTH
      Expiry: "1h" # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDLESSINITCODE_EXPIRY
      IncludeLowerLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDLESSINITCODE_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDLESSINITCODE_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDLESSINITCODE_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_PASSWORDLESSINITCODE_INCLUDESYMBOLS
    DomainVerification:
      Length: 32 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_DOMAINVERIFICATION_LENGTH
      IncludeLowerLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_DOMAINVERIFICATION_INCLUDELOWERLETTERS
      IncludeUpperLetters: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_DOMAINVERIFICATION_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_DOMAINVERIFICATION_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_DOMAINVERIFICATION_INCLUDESYMBOLS
    OTPSMS:
      Length: 8 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPSMS_LENGTH
      Expiry: "5m" # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPSMS_EXPIRY
      IncludeLowerLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPSMS_INCLUDELOWERLETTERS
      IncludeUpperLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPSMS_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPSMS_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPSMS_INCLUDESYMBOLS
    OTPEmail:
      Length: 8 # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPEMAIL_LENGTH
      Expiry: "5m" # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPEMAIL_EXPIRY
      IncludeLowerLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPEMAIL_INCLUDELOWERLETTERS
      IncludeUpperLetters: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPEMAIL_INCLUDEUPPERLETTERS
      IncludeDigits: true # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPEMAIL_INCLUDEDIGITS
      IncludeSymbols: false # ZITADEL_DEFAULTINSTANCE_SECRETGENERATORS_OTPEMAIL_INCLUDESYMBOLS
  PasswordComplexityPolicy:
    MinLength: 8 # ZITADEL_DEFAULTINSTANCE_PASSWORDCOMPLEXITYPOLICY_MINLENGTH
    HasLowercase: true # ZITADEL_DEFAULTINSTANCE_PASSWORDCOMPLEXITYPOLICY_HASLOWERCASE
    HasUppercase: true # ZITADEL_DEFAULTINSTANCE_PASSWORDCOMPLEXITYPOLICY_HASUPPERCASE
    HasNumber: true # ZITADEL_DEFAULTINSTANCE_PASSWORDCOMPLEXITYPOLICY_HASNUMBER
    HasSymbol: true # ZITADEL_DEFAULTINSTANCE_PASSWORDCOMPLEXITYPOLICY_HASSYMBOL
  PasswordAgePolicy:
    ExpireWarnDays: 0 # ZITADEL_DEFAULTINSTANCE_PASSWORDAGEPOLICY_EXPIREWARNDAYS
    MaxAgeDays: 0 # ZITADEL_DEFAULTINSTANCE_PASSWORDAGEPOLICY_MAXAGEDAYS
  DomainPolicy:
    UserLoginMustBeDomain: false # ZITADEL_DEFAULTINSTANCE_DOMAINPOLICY_USERLOGINMUSTBEDOMAIN
    ValidateOrgDomains: false # ZITADEL_DEFAULTINSTANCE_DOMAINPOLICY_VALIDATEORGDOMAINS
    SMTPSenderAddressMatchesInstanceDomain: false # ZITADEL_DEFAULTINSTANCE_DOMAINPOLICY_SMTPSENDERADDRESSMATCHESINSTANCEDOMAIN
  LoginPolicy:
    AllowUsernamePassword: true # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_ALLOWUSERNAMEPASSWORD
    AllowRegister: true # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_ALLOWREGISTER
    AllowExternalIDP: true # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_ALLOWEXTERNALIDP
    ForceMFA: false # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_FORCEMFA
    HidePasswordReset: false # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_HIDEPASSWORDRESET
    IgnoreUnknownUsernames: false # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_IGNOREUNKNOWNUSERNAMES
    AllowDomainDiscovery: true # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_ALLOWDOMAINDISCOVERY
    # 1 is allowed, 0 is not allowed
    PasswordlessType: 1 # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_PASSWORDLESSTYPE
    # DefaultRedirectURL is empty by default because we use the Console UI
    DefaultRedirectURI: # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_DEFAULTREDIRECTURI
    # 240h = 10d
    PasswordCheckLifetime: 240h # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_PASSWORDCHECKLIFETIME
    # 240h = 10d
    ExternalLoginCheckLifetime: 240h # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_EXTERNALLOGINCHECKLIFETIME
    # 720h = 30d
    MfaInitSkipLifetime: 720h # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_MFAINITSKIPLIFETIME
    SecondFactorCheckLifetime: 18h # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_SECONDFACTORCHECKLIFETIME
    MultiFactorCheckLifetime: 12h # ZITADEL_DEFAULTINSTANCE_LOGINPOLICY_MULTIFACTORCHECKLIFETIME
  PrivacyPolicy:
    TOSLink: https://zitadel.com/docs/legal/terms-of-service # ZITADEL_DEFAULTINSTANCE_PRIVACYPOLICY_TOSLINK
    PrivacyLink: https://zitadel.com/docs/legal/privacy-policy # ZITADEL_DEFAULTINSTANCE_PRIVACYPOLICY_PRIVACYLINK
    HelpLink: "" # ZITADEL_DEFAULTINSTANCE_PRIVACYPOLICY_HELPLINK
    SupportEmail: "" # ZITADEL_DEFAULTINSTANCE_PRIVACYPOLICY_SUPPORTEMAIL
    DocsLink: https://zitadel.com/docs # ZITADEL_DEFAULTINSTANCE_PRIVACYPOLICY_DOCSLINK
    CustomLink: "" # ZITADEL_DEFAULTINSTANCE_PRIVACYPOLICY_CUSTOMLINK
    CustomLinkText: "" # ZITADEL_DEFAULTINSTANCE_PRIVACYPOLICY_CUSTOMLINKTEXT
  NotificationPolicy:
    PasswordChange: true # ZITADEL_DEFAULTINSTANCE_NOTIFICATIONPOLICY_PASSWORDCHANGE
  LabelPolicy:
    PrimaryColor: "#5469d4" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_PRIMARYCOLOR
    BackgroundColor: "#fafafa" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_BACKGROUNDCOLOR
    WarnColor: "#cd3d56" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_WARNCOLOR
    FontColor: "#000000" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_FONTCOLOR
    PrimaryColorDark: "#2073c4" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_PRIMARYCOLORDARK
    BackgroundColorDark: "#111827" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_BACKGROUNDCOLORDARK
    WarnColorDark: "#ff3b5b" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_WARNCOLORDARK
    FontColorDark: "#ffffff" # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_FONTCOLORDARK
    HideLoginNameSuffix: false # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_HIDELOGINNAMESUFFIX
    ErrorMsgPopup: false # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_ERRORMSGPOPUP
    DisableWatermark: false # ZITADEL_DEFAULTINSTANCE_LABELPOLICY_DISABLEWATERMARK
  LockoutPolicy:
    MaxPasswordAttempts: 0 # ZITADEL_DEFAULTINSTANCE_LOCKOUTPOLICY_MAXPASSWORDATTEMPTS
    MaxOTPAttempts: 0 # ZITADEL_DEFAULTINSTANCE_LOCKOUTPOLICY_MAXOTPATTEMPTS
    ShouldShowLockoutFailure: true # ZITADEL_DEFAULTINSTANCE_LOCKOUTPOLICY_SHOULDSHOWLOCKOUTFAILURE
  EmailTemplate: CjwhZG9jdHlwZSBodG1sPgo8aHRtbCB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMTk5OS94aHRtbCIgeG1sbnM6dj0idXJuOnNjaGVtYXMtbWljcm9zb2Z0LWNvbTp2bWwiIHhtbG5zOm89InVybjpzY2hlbWFzLW1pY3Jvc29mdC1jb206b2ZmaWNlOm9mZmljZSI+CjxoZWFkPgogIDx0aXRsZT4KCiAgPC90aXRsZT4KICA8IS0tW2lmICFtc29dPjwhLS0+CiAgPG1ldGEgaHR0cC1lcXVpdj0iWC1VQS1Db21wYXRpYmxlIiBjb250ZW50PSJJRT1lZGdlIj4KICA8IS0tPCFbZW5kaWZdLS0+CiAgPG1ldGEgaHR0cC1lcXVpdj0iQ29udGVudC1UeXBlIiBjb250ZW50PSJ0ZXh0L2h0bWw7IGNoYXJzZXQ9VVRGLTgiPgogIDxtZXRhIG5hbWU9InZpZXdwb3J0IiBjb250ZW50PSJ3aWR0aD1kZXZpY2Utd2lkdGgsIGluaXRpYWwtc2NhbGU9MSI+CiAgPHN0eWxlIHR5cGU9InRleHQvY3NzIj4KICAgICNvdXRsb29rIGEgeyBwYWRkaW5nOjA7IH0KICAgIGJvZHkgeyBtYXJnaW46MDtwYWRkaW5nOjA7LXdlYmtpdC10ZXh0LXNpemUtYWRqdXN0OjEwMCU7LW1zLXRleHQtc2l6ZS1hZGp1c3Q6MTAwJTsgfQogICAgdGFibGUsIHRkIHsgYm9yZGVyLWNvbGxhcHNlOmNvbGxhcHNlO21zby10YWJsZS1sc3BhY2U6MHB0O21zby10YWJsZS1yc3BhY2U6MHB0OyB9CiAgICBpbWcgeyBib3JkZXI6MDtoZWlnaHQ6YXV0bztsaW5lLWhlaWdodDoxMDAlOyBvdXRsaW5lOm5vbmU7dGV4dC1kZWNvcmF0aW9uOm5vbmU7LW1zLWludGVycG9sYXRpb24tbW9kZTpiaWN1YmljOyB9CiAgICBwIHsgZGlzcGxheTpibG9jazttYXJnaW46MTNweCAwOyB9CiAgPC9zdHlsZT4KICA8IS0tW2lmIG1zb10+CiAgPHhtbD4KICAgIDxvOk9mZmljZURvY3VtZW50U2V0dGluZ3M+CiAgICAgIDxvOkFsbG93UE5HLz4KICAgICAgPG86UGl4ZWxzUGVySW5jaD45NjwvbzpQaXhlbHNQZXJJbmNoPgogICAgPC9vOk9mZmljZURvY3VtZW50U2V0dGluZ3M+CiAgPC94bWw+CiAgPCFbZW5kaWZdLS0+CiAgPCEtLVtpZiBsdGUgbXNvIDExXT4KICA8c3R5bGUgdHlwZT0idGV4dC9jc3MiPgogICAgLm1qLW91dGxvb2stZ3JvdXAtZml4IHsgd2lkdGg6MTAwJSAhaW1wb3J0YW50OyB9CiAgPC9zdHlsZT4KICA8IVtlbmRpZl0tLT4KCgogIDxzdHlsZSB0eXBlPSJ0ZXh0L2NzcyI+CiAgICBAbWVkaWEgb25seSBzY3JlZW4gYW5kIChtaW4td2lkdGg6NDgwcHgpIHsKICAgICAgLm1qLWNvbHVtbi1wZXItMTAwIHsgd2lkdGg6MTAwJSAhaW1wb3J0YW50OyBtYXgtd2lkdGg6IDEwMCU7IH0KICAgICAgLm1qLWNvbHVtbi1wZXItNjAgeyB3aWR0aDo2MCUgIWltcG9ydGFudDsgbWF4LXdpZHRoOiA2MCU7IH0KICAgIH0KICA8L3N0eWxlPgoKCiAgPHN0eWxlIHR5cGU9InRleHQvY3NzIj4KCgoKICAgIEBtZWRpYSBvbmx5IHNjcmVlbiBhbmQgKG1heC13aWR0aDo0ODBweCkgewogICAgICB0YWJsZS5tai1mdWxsLXdpZHRoLW1vYmlsZSB7IHdpZHRoOiAxMDAlICFpbXBvcnRhbnQ7IH0KICAgICAgdGQubWotZnVsbC13aWR0aC1tb2JpbGUgeyB3aWR0aDogYXV0byAhaW1wb3J0YW50OyB9CiAgICB9CgogIDwvc3R5bGU+CiAgPHN0eWxlIHR5cGU9InRleHQvY3NzIj4uc2hhZG93IGEgewogICAgYm94LXNoYWRvdzogMHB4IDNweCAxcHggLTJweCByZ2JhKDAsIDAsIDAsIDAuMiksIDBweCAycHggMnB4IDBweCByZ2JhKDAsIDAsIDAsIDAuMTQpLCAwcHggMXB4IDVweCAwcHggcmdiYSgwLCAwLCAwLCAwLjEyKTsKICB9PC9zdHlsZT4KCiAge3tpZiAuRm9udFVSTH19CiAgPHN0eWxlPgogICAgQGZvbnQtZmFjZSB7CiAgICAgIGZvbnQtZmFtaWx5OiAne3suRm9udEZhY2VGYW1pbHl9fSc7CiAgICAgIGZvbnQtc3R5bGU6IG5vcm1hbDsKICAgICAgZm9udC1kaXNwbGF5OiBzd2FwOwogICAgICBzcmM6IHVybCh7ey5Gb250VVJMfX0pOwogICAgfQogIDwvc3R5bGU+CiAge3tlbmR9fQoKPC9oZWFkPgo8Ym9keSBzdHlsZT0id29yZC1zcGFjaW5nOm5vcm1hbDsiPgoKCjxkaXYKICAgICAgICBzdHlsZT0iIgo+CgogIDx0YWJsZQogICAgICAgICAgYWxpZ249ImNlbnRlciIgYm9yZGVyPSIwIiBjZWxscGFkZGluZz0iMCIgY2VsbHNwYWNpbmc9IjAiIHJvbGU9InByZXNlbnRhdGlvbiIgc3R5bGU9ImJhY2tncm91bmQ6e3suQmFja2dyb3VuZENvbG9yfX07YmFja2dyb3VuZC1jb2xvcjp7ey5CYWNrZ3JvdW5kQ29sb3J9fTt3aWR0aDoxMDAlO2JvcmRlci1yYWRpdXM6MTZweDsiCiAgPgogICAgPHRib2R5PgogICAgPHRyPgogICAgICA8dGQ+CgoKICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48dGFibGUgYWxpZ249ImNlbnRlciIgYm9yZGVyPSIwIiBjZWxscGFkZGluZz0iMCIgY2VsbHNwYWNpbmc9IjAiIGNsYXNzPSIiIHN0eWxlPSJ3aWR0aDo4MDBweDsiIHdpZHRoPSI4MDAiID48dHI+PHRkIHN0eWxlPSJsaW5lLWhlaWdodDowcHg7Zm9udC1zaXplOjBweDttc28tbGluZS1oZWlnaHQtcnVsZTpleGFjdGx5OyI+PCFbZW5kaWZdLS0+CgoKICAgICAgICA8ZGl2ICBzdHlsZT0ibWFyZ2luOjBweCBhdXRvO2JvcmRlci1yYWRpdXM6MTZweDttYXgtd2lkdGg6ODAwcHg7Ij4KCiAgICAgICAgICA8dGFibGUKICAgICAgICAgICAgICAgICAgYWxpZ249ImNlbnRlciIgYm9yZGVyPSIwIiBjZWxscGFkZGluZz0iMCIgY2VsbHNwYWNpbmc9IjAiIHJvbGU9InByZXNlbnRhdGlvbiIgc3R5bGU9IndpZHRoOjEwMCU7Ym9yZGVyLXJhZGl1czoxNnB4OyIKICAgICAgICAgID4KICAgICAgICAgICAgPHRib2R5PgogICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgPHRkCiAgICAgICAgICAgICAgICAgICAgICBzdHlsZT0iZGlyZWN0aW9uOmx0cjtmb250LXNpemU6MHB4O3BhZGRpbmc6MjBweCAwO3BhZGRpbmctbGVmdDowO3RleHQtYWxpZ246Y2VudGVyOyIKICAgICAgICAgICAgICA+CiAgICAgICAgICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48dGFibGUgcm9sZT0icHJlc2VudGF0aW9uIiBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCI+PHRyPjx0ZCBjbGFzcz0iIiB3aWR0aD0iODAwcHgiID48IVtlbmRpZl0tLT4KCiAgICAgICAgICAgICAgICA8dGFibGUKICAgICAgICAgICAgICAgICAgICAgICAgYWxpZ249ImNlbnRlciIgYm9yZGVyPSIwIiBjZWxscGFkZGluZz0iMCIgY2VsbHNwYWNpbmc9IjAiIHJvbGU9InByZXNlbnRhdGlvbiIgc3R5bGU9IndpZHRoOjEwMCU7IgogICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICA8dGJvZHk+CiAgICAgICAgICAgICAgICAgIDx0cj4KICAgICAgICAgICAgICAgICAgICA8dGQ+CgoKICAgICAgICAgICAgICAgICAgICAgIDwhLS1baWYgbXNvIHwgSUVdPjx0YWJsZSBhbGlnbj0iY2VudGVyIiBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgY2xhc3M9IiIgc3R5bGU9IndpZHRoOjgwMHB4OyIgd2lkdGg9IjgwMCIgPjx0cj48dGQgc3R5bGU9ImxpbmUtaGVpZ2h0OjBweDtmb250LXNpemU6MHB4O21zby1saW5lLWhlaWdodC1ydWxlOmV4YWN0bHk7Ij48IVtlbmRpZl0tLT4KCgogICAgICAgICAgICAgICAgICAgICAgPGRpdiAgc3R5bGU9Im1hcmdpbjowcHggYXV0bzttYXgtd2lkdGg6ODAwcHg7Ij4KCiAgICAgICAgICAgICAgICAgICAgICAgIDx0YWJsZQogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGFsaWduPSJjZW50ZXIiIGJvcmRlcj0iMCIgY2VsbHBhZGRpbmc9IjAiIGNlbGxzcGFjaW5nPSIwIiByb2xlPSJwcmVzZW50YXRpb24iIHN0eWxlPSJ3aWR0aDoxMDAlOyIKICAgICAgICAgICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICAgICAgICAgIDx0Ym9keT4KICAgICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dGQKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgc3R5bGU9ImRpcmVjdGlvbjpsdHI7Zm9udC1zaXplOjBweDtwYWRkaW5nOjA7dGV4dC1hbGlnbjpjZW50ZXI7IgogICAgICAgICAgICAgICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48dGFibGUgcm9sZT0icHJlc2VudGF0aW9uIiBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCI+PHRyPjx0ZCBjbGFzcz0iIiBzdHlsZT0id2lkdGg6ODAwcHg7IiA+PCFbZW5kaWZdLS0+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8ZGl2CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgY2xhc3M9Im1qLWNvbHVtbi1wZXItMTAwIG1qLW91dGxvb2stZ3JvdXAtZml4IiBzdHlsZT0iZm9udC1zaXplOjA7bGluZS1oZWlnaHQ6MDt0ZXh0LWFsaWduOmxlZnQ7ZGlzcGxheTppbmxpbmUtYmxvY2s7d2lkdGg6MTAwJTtkaXJlY3Rpb246bHRyOyIKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwhLS1baWYgbXNvIHwgSUVdPjx0YWJsZSBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgcm9sZT0icHJlc2VudGF0aW9uIiA+PHRyPjx0ZCBzdHlsZT0idmVydGljYWwtYWxpZ246dG9wO3dpZHRoOjgwMHB4OyIgPjwhW2VuZGlmXS0tPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8ZGl2CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBjbGFzcz0ibWotY29sdW1uLXBlci0xMDAgbWotb3V0bG9vay1ncm91cC1maXgiIHN0eWxlPSJmb250LXNpemU6MHB4O3RleHQtYWxpZ246bGVmdDtkaXJlY3Rpb246bHRyO2Rpc3BsYXk6aW5saW5lLWJsb2NrO3ZlcnRpY2FsLWFsaWduOnRvcDt3aWR0aDoxMDAlOyIKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRhYmxlCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGJvcmRlcj0iMCIgY2VsbHBhZGRpbmc9IjAiIGNlbGxzcGFjaW5nPSIwIiByb2xlPSJwcmVzZW50YXRpb24iIHdpZHRoPSIxMDAlIgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dGJvZHk+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0cj4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dGQgIHN0eWxlPSJ2ZXJ0aWNhbC1hbGlnbjp0b3A7cGFkZGluZzowOyI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB7e2lmIC5Mb2dvVVJMfX0KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0YWJsZQogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgcm9sZT0icHJlc2VudGF0aW9uIiBzdHlsZT0iIiB3aWR0aD0iMTAwJSIKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRib2R5PgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRyPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0ZAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgYWxpZ249ImNlbnRlciIgc3R5bGU9ImZvbnQtc2l6ZTowcHg7cGFkZGluZzo1MHB4IDAgMzBweCAwO3dvcmQtYnJlYWs6YnJlYWstd29yZDsiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0YWJsZQogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgcm9sZT0icHJlc2VudGF0aW9uIiBzdHlsZT0iYm9yZGVyLWNvbGxhcHNlOmNvbGxhcHNlO2JvcmRlci1zcGFjaW5nOjBweDsiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0Ym9keT4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRyPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0ZCAgc3R5bGU9IndpZHRoOjE4MHB4OyI+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPGltZwogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBoZWlnaHQ9ImF1dG8iIHNyYz0ie3suTG9nb1VSTH19IiBzdHlsZT0iYm9yZGVyOjA7Ym9yZGVyLXJhZGl1czo4cHg7ZGlzcGxheTpibG9jaztvdXRsaW5lOm5vbmU7dGV4dC1kZWNvcmF0aW9uOm5vbmU7aGVpZ2h0OmF1dG87d2lkdGg6MTAwJTtmb250LXNpemU6MTNweDsiIHdpZHRoPSIxODAiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAvPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RkPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RyPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3Rib2R5PgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90YWJsZT4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGJvZHk+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RhYmxlPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAge3tlbmR9fQogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGJvZHk+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RhYmxlPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L2Rpdj4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPCEtLVtpZiBtc28gfCBJRV0+PC90ZD48L3RyPjwvdGFibGU+PCFbZW5kaWZdLS0+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvZGl2PgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPCEtLVtpZiBtc28gfCBJRV0+PC90ZD48L3RyPjwvdGFibGU+PCFbZW5kaWZdLS0+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RkPgogICAgICAgICAgICAgICAgICAgICAgICAgIDwvdHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgPC90Ym9keT4KICAgICAgICAgICAgICAgICAgICAgICAgPC90YWJsZT4KCiAgICAgICAgICAgICAgICAgICAgICA8L2Rpdj4KCgogICAgICAgICAgICAgICAgICAgICAgPCEtLVtpZiBtc28gfCBJRV0+PC90ZD48L3RyPjwvdGFibGU+PCFbZW5kaWZdLS0+CgoKICAgICAgICAgICAgICAgICAgICA8L3RkPgogICAgICAgICAgICAgICAgICA8L3RyPgogICAgICAgICAgICAgICAgICA8L3Rib2R5PgogICAgICAgICAgICAgICAgPC90YWJsZT4KCiAgICAgICAgICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48L3RkPjwvdHI+PHRyPjx0ZCBjbGFzcz0iIiB3aWR0aD0iODAwcHgiID48IVtlbmRpZl0tLT4KCiAgICAgICAgICAgICAgICA8dGFibGUKICAgICAgICAgICAgICAgICAgICAgICAgYWxpZ249ImNlbnRlciIgYm9yZGVyPSIwIiBjZWxscGFkZGluZz0iMCIgY2VsbHNwYWNpbmc9IjAiIHJvbGU9InByZXNlbnRhdGlvbiIgc3R5bGU9IndpZHRoOjEwMCU7IgogICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICA8dGJvZHk+CiAgICAgICAgICAgICAgICAgIDx0cj4KICAgICAgICAgICAgICAgICAgICA8dGQ+CgoKICAgICAgICAgICAgICAgICAgICAgIDwhLS1baWYgbXNvIHwgSUVdPjx0YWJsZSBhbGlnbj0iY2VudGVyIiBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgY2xhc3M9IiIgc3R5bGU9IndpZHRoOjgwMHB4OyIgd2lkdGg9IjgwMCIgPjx0cj48dGQgc3R5bGU9ImxpbmUtaGVpZ2h0OjBweDtmb250LXNpemU6MHB4O21zby1saW5lLWhlaWdodC1ydWxlOmV4YWN0bHk7Ij48IVtlbmRpZl0tLT4KCgogICAgICAgICAgICAgICAgICAgICAgPGRpdiAgc3R5bGU9Im1hcmdpbjowcHggYXV0bzttYXgtd2lkdGg6ODAwcHg7Ij4KCiAgICAgICAgICAgICAgICAgICAgICAgIDx0YWJsZQogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGFsaWduPSJjZW50ZXIiIGJvcmRlcj0iMCIgY2VsbHBhZGRpbmc9IjAiIGNlbGxzcGFjaW5nPSIwIiByb2xlPSJwcmVzZW50YXRpb24iIHN0eWxlPSJ3aWR0aDoxMDAlOyIKICAgICAgICAgICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICAgICAgICAgIDx0Ym9keT4KICAgICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dGQKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgc3R5bGU9ImRpcmVjdGlvbjpsdHI7Zm9udC1zaXplOjBweDtwYWRkaW5nOjA7dGV4dC1hbGlnbjpjZW50ZXI7IgogICAgICAgICAgICAgICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48dGFibGUgcm9sZT0icHJlc2VudGF0aW9uIiBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCI+PHRyPjx0ZCBjbGFzcz0iIiBzdHlsZT0idmVydGljYWwtYWxpZ246dG9wO3dpZHRoOjQ4MHB4OyIgPjwhW2VuZGlmXS0tPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPGRpdgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGNsYXNzPSJtai1jb2x1bW4tcGVyLTYwIG1qLW91dGxvb2stZ3JvdXAtZml4IiBzdHlsZT0iZm9udC1zaXplOjBweDt0ZXh0LWFsaWduOmxlZnQ7ZGlyZWN0aW9uOmx0cjtkaXNwbGF5OmlubGluZS1ibG9jazt2ZXJ0aWNhbC1hbGlnbjp0b3A7d2lkdGg6MTAwJTsiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRhYmxlCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgcm9sZT0icHJlc2VudGF0aW9uIiB3aWR0aD0iMTAwJSIKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dGJvZHk+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0ZCAgc3R5bGU9InZlcnRpY2FsLWFsaWduOnRvcDtwYWRkaW5nOjA7Ij4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRhYmxlCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgcm9sZT0icHJlc2VudGF0aW9uIiBzdHlsZT0iIiB3aWR0aD0iMTAwJSIKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dGJvZHk+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRyPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dGQKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBhbGlnbj0iY2VudGVyIiBzdHlsZT0iZm9udC1zaXplOjBweDtwYWRkaW5nOjEwcHggMjVweDt3b3JkLWJyZWFrOmJyZWFrLXdvcmQ7IgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDxkaXYKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIHN0eWxlPSJmb250LWZhbWlseTp7ey5Gb250RmFtaWx5fX07Zm9udC1zaXplOjI0cHg7Zm9udC13ZWlnaHQ6NTAwO2xpbmUtaGVpZ2h0OjE7dGV4dC1hbGlnbjpjZW50ZXI7Y29sb3I6e3suRm9udENvbG9yfX07IgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID57ey5HcmVldGluZ319PC9kaXY+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RkPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0ZAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGFsaWduPSJjZW50ZXIiIHN0eWxlPSJmb250LXNpemU6MHB4O3BhZGRpbmc6MTBweCAyNXB4O3dvcmQtYnJlYWs6YnJlYWstd29yZDsiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPGRpdgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgc3R5bGU9ImZvbnQtZmFtaWx5Ont7LkZvbnRGYW1pbHl9fTtmb250LXNpemU6MTZweDtmb250LXdlaWdodDpsaWdodDtsaW5lLWhlaWdodDoxLjU7dGV4dC1hbGlnbjpjZW50ZXI7Y29sb3I6e3suRm9udENvbG9yfX07IgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID57ey5UZXh0fX08L2Rpdj4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RyPgoKCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8dHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0ZAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGFsaWduPSJjZW50ZXIiIHZlcnRpY2FsLWFsaWduPSJtaWRkbGUiIGNsYXNzPSJzaGFkb3ciIHN0eWxlPSJmb250LXNpemU6MHB4O3BhZGRpbmc6MTBweCAyNXB4O3dvcmQtYnJlYWs6YnJlYWstd29yZDsiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRhYmxlCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBib3JkZXI9IjAiIGNlbGxwYWRkaW5nPSIwIiBjZWxsc3BhY2luZz0iMCIgcm9sZT0icHJlc2VudGF0aW9uIiBzdHlsZT0iYm9yZGVyLWNvbGxhcHNlOnNlcGFyYXRlO2xpbmUtaGVpZ2h0OjEwMCU7IgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0cj4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRkCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgYWxpZ249ImNlbnRlciIgYmdjb2xvcj0ie3suUHJpbWFyeUNvbG9yfX0iIHJvbGU9InByZXNlbnRhdGlvbiIgc3R5bGU9ImJvcmRlcjpub25lO2JvcmRlci1yYWRpdXM6NnB4O2N1cnNvcjphdXRvO21zby1wYWRkaW5nLWFsdDoxMHB4IDI1cHg7YmFja2dyb3VuZDp7ey5QcmltYXJ5Q29sb3J9fTsiIHZhbGlnbj0ibWlkZGxlIgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPGEKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIGhyZWY9Int7LlVSTH19IiByZWw9Im5vb3BlbmVyIG5vcmVmZXJyZXIgbm90cmFjayIgc3R5bGU9ImRpc3BsYXk6aW5saW5lLWJsb2NrO2JhY2tncm91bmQ6e3suUHJpbWFyeUNvbG9yfX07Y29sb3I6I2ZmZmZmZjtmb250LWZhbWlseTp7ey5Gb250RmFtaWx5fX07Zm9udC1zaXplOjE0cHg7Zm9udC13ZWlnaHQ6NTAwO2xpbmUtaGVpZ2h0OjEyMCU7bWFyZ2luOjA7dGV4dC1kZWNvcmF0aW9uOm5vbmU7dGV4dC10cmFuc2Zvcm06bm9uZTtwYWRkaW5nOjEwcHggMjVweDttc28tcGFkZGluZy1hbHQ6MHB4O2JvcmRlci1yYWRpdXM6NnB4OyIgdGFyZ2V0PSJfYmxhbmsiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAge3suQnV0dG9uVGV4dH19CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC9hPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RkPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RhYmxlPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90ZD4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdHI+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB7e2lmIC5JbmNsdWRlRm9vdGVyfX0KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0cj4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRkCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgYWxpZ249ImNlbnRlciIgc3R5bGU9ImZvbnQtc2l6ZTowcHg7cGFkZGluZzoxMHB4IDI1cHg7cGFkZGluZy10b3A6MjBweDtwYWRkaW5nLXJpZ2h0OjIwcHg7cGFkZGluZy1ib3R0b206MjBweDtwYWRkaW5nLWxlZnQ6MjBweDt3b3JkLWJyZWFrOmJyZWFrLXdvcmQ7IgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDxwCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICBzdHlsZT0iYm9yZGVyLXRvcDpzb2xpZCAycHggI2RiZGJkYjtmb250LXNpemU6MXB4O21hcmdpbjowcHggYXV0bzt3aWR0aDoxMDAlOyIKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC9wPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48dGFibGUgYWxpZ249ImNlbnRlciIgYm9yZGVyPSIwIiBjZWxscGFkZGluZz0iMCIgY2VsbHNwYWNpbmc9IjAiIHN0eWxlPSJib3JkZXItdG9wOnNvbGlkIDJweCAjZGJkYmRiO2ZvbnQtc2l6ZToxcHg7bWFyZ2luOjBweCBhdXRvO3dpZHRoOjQ0MHB4OyIgcm9sZT0icHJlc2VudGF0aW9uIiB3aWR0aD0iNDQwcHgiID48dHI+PHRkIHN0eWxlPSJoZWlnaHQ6MDtsaW5lLWhlaWdodDowOyI+ICZuYnNwOwogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGQ+PC90cj48L3RhYmxlPjwhW2VuZGlmXS0tPgoKCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RyPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDx0cj4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPHRkCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgYWxpZ249ImNlbnRlciIgc3R5bGU9ImZvbnQtc2l6ZTowcHg7cGFkZGluZzoxNnB4O3dvcmQtYnJlYWs6YnJlYWstd29yZDsiCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgID4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPGRpdgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgc3R5bGU9ImZvbnQtZmFtaWx5Ont7LkZvbnRGYW1pbHl9fTtmb250LXNpemU6MTNweDtsaW5lLWhlaWdodDoxO3RleHQtYWxpZ246Y2VudGVyO2NvbG9yOnt7LkZvbnRDb2xvcn19OyIKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA+e3suRm9vdGVyVGV4dH19PC9kaXY+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RkPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIHt7ZW5kfX0KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGJvZHk+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90YWJsZT4KCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RyPgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC90Ym9keT4KICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8L3RhYmxlPgoKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgPC9kaXY+CgogICAgICAgICAgICAgICAgICAgICAgICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48L3RkPjwvdHI+PC90YWJsZT48IVtlbmRpZl0tLT4KICAgICAgICAgICAgICAgICAgICAgICAgICAgIDwvdGQ+CiAgICAgICAgICAgICAgICAgICAgICAgICAgPC90cj4KICAgICAgICAgICAgICAgICAgICAgICAgICA8L3Rib2R5PgogICAgICAgICAgICAgICAgICAgICAgICA8L3RhYmxlPgoKICAgICAgICAgICAgICAgICAgICAgIDwvZGl2PgoKCiAgICAgICAgICAgICAgICAgICAgICA8IS0tW2lmIG1zbyB8IElFXT48L3RkPjwvdHI+PC90YWJsZT48IVtlbmRpZl0tLT4KCgogICAgICAgICAgICAgICAgICAgIDwvdGQ+CiAgICAgICAgICAgICAgICAgIDwvdHI+CiAgICAgICAgICAgICAgICAgIDwvdGJvZHk+CiAgICAgICAgICAgICAgICA8L3RhYmxlPgoKICAgICAgICAgICAgICAgIDwhLS1baWYgbXNvIHwgSUVdPjwvdGQ+PC90cj48L3RhYmxlPjwhW2VuZGlmXS0tPgogICAgICAgICAgICAgIDwvdGQ+CiAgICAgICAgICAgIDwvdHI+CiAgICAgICAgICAgIDwvdGJvZHk+CiAgICAgICAgICA8L3RhYmxlPgoKICAgICAgICA8L2Rpdj4KCgogICAgICAgIDwhLS1baWYgbXNvIHwgSUVdPjwvdGQ+PC90cj48L3RhYmxlPjwhW2VuZGlmXS0tPgoKCiAgICAgIDwvdGQ+CiAgICA8L3RyPgogICAgPC90Ym9keT4KICA8L3RhYmxlPgoKPC9kaXY+Cgo8L2JvZHk+CjwvaHRtbD4K # ZITADEL_DEFAULTINSTANCE_EMAILTEMPLATE
  # Sets the default values for lifetime and expiration for OIDC in each newly created instance
  # This default can be overwritten for each instance during runtime
  # Overwrites the system defaults
  # If defined but not all durations are set it will result in an error
  OIDCSettings:
    AccessTokenLifetime: 12h # ZITADEL_DEFAULTINSTANCE_OIDCSETTINGS_ACCESSTOKENLIFETIME
    IdTokenLifetime: 12h # ZITADEL_DEFAULTINSTANCE_OIDCSETTINGS_IDTOKENLIFETIME
    # 720h are 30 days
    RefreshTokenIdleExpiration: 720h # ZITADEL_DEFAULTINSTANCE_OIDCSETTINGS_REFRESHTOKENIDLEEXPIRATION
    # 2160h are 90 days
    RefreshTokenExpiration: 2160h # ZITADEL_DEFAULTINSTANCE_OIDCSETTINGS_REFRESHTOKENEXPIRATION
  # this configuration sets the default email configuration
  SMTPConfiguration:
    # Configuration of the host
    SMTP:
      # must include the port, like smtp.mailtrap.io:2525. IPv6 is also supported, like [2001:db8::1]:2525
      Host: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_HOST
      User: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_USER
      Password: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_SMTP_PASSWORD
    TLS: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_TLS
    # If the host of the sender is different from ExternalDomain set DefaultInstance.DomainPolicy.SMTPSenderAddressMatchesInstanceDomain to false
    From: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_FROM
    FromName: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_FROMNAME
    ReplyToAddress: # ZITADEL_DEFAULTINSTANCE_SMTPCONFIGURATION_REPLYTOADDRESS
  # Configure the MessageTexts by environment variable using JSON notation:
  # ZITADEL_DEFAULTINSTANCE_MESSAGETEXTS='[{"messageTextType": "InitCode", "title": "My custom title"},{"messageTextType": "PasswordReset", "greeting": "Hi there!"}]'
  # Beware that if you configure the MessageTexts by environment variable, all the default MessageTexts are lost.
  MessageTexts:
    - MessageTextType: InitCode
      Language: de
      Title: Zitadel - User initialisieren
      PreHeader: User initialisieren
      Subject: User initialisieren
      Greeting: Hallo {{.DisplayName}},
      Text: Dieser Benutzer wurde soeben im Zitadel erstellt. Mit dem Benutzernamen &lt;br&gt;&lt;strong&gt;{{.PreferredLoginName}}&lt;/strong&gt;&lt;br&gt; kannst du dich anmelden. Nutze den untenstehenden Button, um die Initialisierung abzuschliessen &lt;br&gt;(Code &lt;strong&gt;{{.Code}}&lt;/strong&gt;).&lt;br&gt; Falls du dieses Mail nicht angefordert hast, kannst du es einfach ignorieren.
      ButtonText: Initialisierung abschliessen
    - MessageTextType: PasswordReset
      Language: de
      Title: Zitadel - Passwort zurücksetzen
      PreHeader: Passwort zurücksetzen
      Subject: Passwort zurücksetzen
      Greeting: Hallo {{.DisplayName}},
      Text: Wir haben eine Anfrage für das Zurücksetzen deines Passwortes bekommen. Du kannst den untenstehenden Button verwenden, um dein Passwort zurückzusetzen &lt;br&gt;(Code &lt;strong&gt;{{.Code}}&lt;/strong&gt;).&lt;br&gt; Falls du dieses Mail nicht angefordert hast, kannst du es ignorieren.
      ButtonText: Passwort zurücksetzen
    - MessageTextType: VerifyEmail
      Language: de
      Title: Zitadel - Email verifizieren
      PreHeader: Email verifizieren
      Subject: Email verifizieren
      Greeting: Hallo {{.DisplayName}},
      Text: Eine neue E-Mail Adresse wurde hinzugefügt. Bitte verwende den untenstehenden Button um diese zu verifizieren &lt;br&gt;(Code &lt;strong&gt;{{.Code}}&lt;/strong&gt;).&lt;br&gt; Falls du deine E-Mail Adresse nicht selber hinzugefügt hast, kannst du dieses E-Mail ignorieren.
      ButtonText: Email verifizieren
    - MessageTextType: VerifyPhone
      Language: de
      Title: Zitadel - Telefonnummer verifizieren
      PreHeader: Telefonnummer verifizieren
      Subject: Telefonnummer verifizieren
      Greeting: Hallo {{.DisplayName}},
      Text: Eine Telefonnummer wurde hinzugefügt. Bitte verifiziere diese in dem du folgenden Code eingibst (Code {{.Code}})
      ButtonText: Telefon verifizieren
    - MessageTextType: DomainClaimed
      Language: de
      Title: Zitadel - Domain wurde beansprucht
      PreHeader: Email / Username ändern
      Subject: Domain wurde beansprucht
      Greeting: Hallo {{.DisplayName}},
      Text: Die Domain {{.Domain}} wurde von einer Organisation beansprucht. Dein derzeitiger User {{.Username}} ist nicht Teil dieser Organisation. Daher musst du beim nächsten Login eine neue Email hinterlegen. Für diesen Login haben wir dir einen temporären Usernamen ({{.TempUsername}}) erstellt.
      ButtonText: Login
    - MessageTextType: PasswordChange
      Language: de
      Title: ZITADEL - Passwort von Benutzer wurde geändert
      PreHeader: Passwort Änderung
      Subject: Passwort von Benutzer wurde geändert
      Greeting: Hallo {{.DisplayName}},
      Text: Das Password vom Benutzer wurde geändert. Wenn diese Änderung von jemand anderem gemacht wurde, empfehlen wir die sofortige Zurücksetzung ihres Passworts.
      ButtonText: Login
    - MessageTextType: InitCode
      Language: en
      Title: Zitadel - Initialize User
      PreHeader: Initialize User
      Subject: Initialize User
      Greeting: Hello {{.DisplayName}},
      Text: This user was created in Zitadel. Use the username {{.PreferredLoginName}} to login. Please click the button below to finish the initialization process. (Code {{.Code}}) If you didn't ask for this mail, please ignore it.
      ButtonText: Finish initialization
    - MessageTextType: PasswordReset
      Language: en
      Title: Zitadel - Reset password
      PreHeader: Reset password
      Subject: Reset password
      Greeting: Hello {{.DisplayName}},
      Text: We received a password reset request. Please use the button below to reset your password. (Code {{.Code}}) If you didn't ask for this mail, please ignore it.
      ButtonText: Reset password
    - MessageTextType: VerifyEmail
      Language: en
      Title: Zitadel - Verify email
      PreHeader: Verify email
      Subject: Verify email
      Greeting: Hello {{.DisplayName}},
      Text: A new email has been added. Please use the button below to verify your email. (Code {{.Code}}) If you din't add a new email, please ignore this email.
      ButtonText: Verify email
    - MessageTextType: VerifyPhone
      Language: en
      Title: Zitadel - Verify phone
      PreHeader: Verify phone
      Subject: Verify phone
      Greeting: Hello {{.DisplayName}},
      Text: A new phone number has been added. Please use the following code to verify it {{.Code}}.
      ButtonText: Verify phone
    - MessageTextType: DomainClaimed
      Language: en
      Title: Zitadel - Domain has been claimed
      PreHeader: Change email/username
      Subject: Domain has been claimed
      Greeting: Hello {{.DisplayName}},
      Text: The domain {{.Domain}} has been claimed by an organization. Your current user {{.UserName}} is not part of this organization. Therefore you'll have to change your email when you login. We have created a temporary username ({{.TempUsername}}) for this login.
      ButtonText: Login
    - MessageTextType: PasswordChange
      Language: en
      Title: ZITADEL - Password of user has changed
      PreHeader: Change password
      Subject: Password of user has changed
      Greeting: Hello {{.DisplayName}},
      Text: The password of your user has changed. If this change was not done by you, please be advised to immediately reset your password.
      ButtonText: Login

  # Once a feature is set on the instance (true or false), system level feature settings
  # will be ignored until instance level features are reset.
  Features:
    LoginDefaultOrg: true # ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINDEFAULTORG
    # TriggerIntrospectionProjections: false # ZITADEL_DEFAULTINSTANCE_FEATURES_TRIGGERINTROSPECTIONPROJECTIONS
    # LegacyIntrospection: false # ZITADEL_DEFAULTINSTANCE_FEATURES_LEGACYINTROSPECTION
  Limits:
    # AuditLogRetention limits the number of events that can be queried via the events API by their age.
    # A value of "0s" means that all events are available.
    # If this value is set, it overwrites the system default unless it is not reset via the admin API.
    AuditLogRetention: # ZITADEL_DEFAULTINSTANCE_LIMITS_AUDITLOGRETENTION
    # If Block is true, all requests except to /ui/console or the system API are blocked and /ui/login is redirected to /ui/console.
    # /ui/console shows a message that the instance is blocked with a link to Console.InstanceManagementURL
    Block: # ZITADEL_DEFAULTINSTANCE_LIMITS_BLOCK
  Restrictions:
    # DisallowPublicOrgRegistration defines if ZITADEL should expose the endpoint /ui/login/register/org
    # If it is true, the endpoint returns the HTTP status 404 on GET requests, and 409 on POST requests.
    DisallowPublicOrgRegistration: # ZITADEL_DEFAULTINSTANCE_RESTRICTIONS_DISALLOWPUBLICORGREGISTRATION
    # AllowedLanguages restricts the languages that can be used.
    # If the list is empty, all supported languages are allowed.
    AllowedLanguages: # ZITADEL_DEFAULTINSTANCE_RESTRICTIONS_ALLOWEDLANGUAGES
    # - en
    # - de
  Quotas:
    # Items take a slice of quota configurations, whereas, for each unit type and instance, one or zero quotas may exist.
    # The following unit types are supported

    # "requests.all.authenticated"
    # The sum of all requests to the ZITADEL API with an authorization header,
    # excluding the following exceptions
    # - Calls to the System API
    # - Calls that cause internal server errors
    # - Failed authorizations
    # - Requests after the quota already exceeded

    # "actions.all.runs.seconds"
    # The sum of all actions run durations in seconds
    # Configure the Items by environment variable using JSON notation:
    # ZITADEL_DEFAULTINSTANCE_QUOTAS_ITEMS='[{"unit": "requests.all.authenticated", "notifications": [{"percent": 100}]}]'
    Items: # ZITADEL_DEFAULTINSTANCE_QUOTAS_ITEMS
#      - Unit: "requests.all.authenticated"
#        # From defines the starting time from which the current quota period is calculated.
#        # This is relevant for querying the current usage.
#        From: "2023-01-01T00:00:00Z"
#        # ResetInterval defines the quota periods duration
#        ResetInterval: 720h # 30 days
#        # Amount defines the number of units for this quota
#        Amount: 25000
#        # Limit defines whether ZITADEL should block further authenticated requests when the configured amount is used.
#        # If you not only want to block authenticated requests but also authentication itself, consider using the system APIs SetLimits method.
#        Limit: false
#        # Notifications are emitted by ZITADEL when certain quota percentages are reached
#        Notifications:
#            # Percent defines the relative amount of used units, after which a notification should be emitted.
#          - Percent: 100
#            # Repeat defines, whether a notification should be emitted each time when a multitude of the configured Percent is used.
#            Repeat: true
#            # CallURL is called when a relative amount of the quota is used.
#            CallURL: "https://httpbin.org/post"

# AuditLogRetention limits the number of events that can be queried via the events API by their age.
# A value of "0s" means that all events are available.
# If an audit log retention is set using an instance limit, it will overwrite the system default.
AuditLogRetention: 0s # ZITADEL_AUDITLOGRETENTION

InternalAuthZ:
  # Configure the RolePermissionMappings by environment variable using JSON notation:
  # ZITADEL_INTERNALAUTHZ_ROLEPERMISSIONMAPPINGS='[{"role": "IAM_OWNER", "permissions": ["iam.write"]}, {"role": "ORG_OWNER", "permissions": ["org.write"]}]'
  # Beware that if you configure the RolePermissionMappings by environment variable, all the default RolePermissionMappings are lost.
  RolePermissionMappings:
    - Role: "SYSTEM_OWNER"
      Permissions:
        - "system.instance.read"
        - "system.instance.write"
        - "system.instance.delete"
        - "system.domain.read"
        - "system.domain.write"
        - "system.domain.delete"
        - "system.debug.read"
        - "system.debug.write"
        - "system.debug.delete"
        - "system.feature.read"
        - "system.feature.write"
        - "system.feature.delete"
        - "system.limits.write"
        - "system.limits.delete"
        - "system.quota.write"
        - "system.quota.delete"
        - "system.iam.member.read"
    - Role: "SYSTEM_OWNER_VIEWER"
      Permissions:
        - "system.instance.read"
        - "system.domain.read"
        - "system.debug.read"
        - "system.feature.read"
        - "system.iam.member.read"
    - Role: "IAM_OWNER"
      Permissions:
        - "iam.read"
        - "iam.write"
        - "iam.policy.read"
        - "iam.policy.write"
        - "iam.policy.delete"
        - "iam.member.read"
        - "iam.member.write"
        - "iam.member.delete"
        - "iam.idp.read"
        - "iam.idp.write"
        - "iam.idp.delete"
        - "iam.action.read"
        - "iam.action.write"
        - "iam.action.delete"
        - "iam.flow.read"
        - "iam.flow.write"
        - "iam.flow.delete"
        - "iam.feature.read"
        - "iam.feature.write"
        - "iam.feature.delete"
        - "iam.restrictions.read"
        - "iam.restrictions.write"
        - "org.read"
        - "org.global.read"
        - "org.create"
        - "org.write"
        - "org.delete"
        - "org.member.read"
        - "org.member.write"
        - "org.member.delete"
        - "org.idp.read"
        - "org.idp.write"
        - "org.idp.delete"
        - "org.action.read"
        - "org.action.write"
        - "org.action.delete"
        - "org.flow.read"
        - "org.flow.write"
        - "org.flow.delete"
        - "org.feature.read"
        - "org.feature.write"
        - "org.feature.delete"
        - "user.read"
        - "user.global.read"
        - "user.write"
        - "user.delete"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
        - "user.credential.write"
        - "user.passkey.write"
        - "user.feature.read"
        - "user.feature.write"
        - "user.feature.delete"
        - "policy.read"
        - "policy.write"
        - "policy.delete"
        - "project.read"
        - "project.create"
        - "project.write"
        - "project.delete"
        - "project.member.read"
        - "project.member.write"
        - "project.member.delete"
        - "project.role.read"
        - "project.role.write"
        - "project.role.delete"
        - "project.app.read"
        - "project.app.write"
        - "project.app.delete"
        - "project.grant.read"
        - "project.grant.write"
        - "project.grant.delete"
        - "project.grant.member.read"
        - "project.grant.member.write"
        - "project.grant.member.delete"
        - "events.read"
        - "milestones.read"
        - "session.delete"
        - "execution.target.read"
        - "execution.target.write"
        - "execution.target.delete"
        - "execution.read"
        - "execution.write"
        - "execution.delete"
        - "userschema.read"
        - "userschema.write"
        - "userschema.delete"
    - Role: "IAM_OWNER_VIEWER"
      Permissions:
        - "iam.read"
        - "iam.policy.read"
        - "iam.member.read"
        - "iam.idp.read"
        - "iam.action.read"
        - "iam.flow.read"
        - "iam.restrictions.read"
        - "iam.feature.read"
        - "org.read"
        - "org.member.read"
        - "org.idp.read"
        - "org.action.read"
        - "org.flow.read"
        - "org.feature.read"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.membership.read"
        - "user.feature.read"
        - "policy.read"
        - "project.read"
        - "project.member.read"
        - "project.role.read"
        - "project.app.read"
        - "project.grant.read"
        - "project.grant.member.read"
        - "events.read"
        - "milestones.read"
        - "execution.target.read"
        - "execution.read"
        - "userschema.read"
    - Role: "IAM_ORG_MANAGER"
      Permissions:
        - "org.read"
        - "org.global.read"
        - "org.create"
        - "org.write"
        - "org.delete"
        - "org.member.read"
        - "org.member.write"
        - "org.member.delete"
        - "org.idp.read"
        - "org.idp.write"
        - "org.idp.delete"
        - "org.action.read"
        - "org.action.write"
        - "org.action.delete"
        - "org.flow.read"
        - "org.flow.write"
        - "org.flow.delete"
        - "org.feature.read"
        - "org.feature.write"
        - "org.feature.delete"
        - "user.read"
        - "user.global.read"
        - "user.write"
        - "user.delete"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
        - "user.credential.write"
        - "user.passkey.write"
        - "user.feature.read"
        - "user.feature.write"
        - "user.feature.delete"
        - "policy.read"
        - "policy.write"
        - "policy.delete"
        - "project.read"
        - "project.create"
        - "project.write"
        - "project.delete"
        - "project.member.read"
        - "project.member.write"
        - "project.member.delete"
        - "project.role.read"
        - "project.role.write"
        - "project.role.delete"
        - "project.app.read"
        - "project.app.write"
        - "project.app.delete"
        - "project.grant.read"
        - "project.grant.write"
        - "project.grant.delete"
        - "project.grant.member.read"
        - "project.grant.member.write"
        - "project.grant.member.delete"
        - "session.delete"
    - Role: "IAM_USER_MANAGER"
      Permissions:
        - "org.read"
        - "org.global.read"
        - "org.member.read"
        - "org.member.delete"
        - "user.read"
        - "user.global.read"
        - "user.write"
        - "user.delete"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
        - "user.passkey.write"
        - "user.feature.read"
        - "user.feature.write"
        - "user.feature.delete"
        - "project.read"
        - "project.member.read"
        - "project.role.read"
        - "project.app.read"
        - "project.grant.read"
        - "project.grant.write"
        - "project.grant.delete"
        - "project.grant.member.read"
        - "session.delete"
    - Role: "IAM_ADMIN_IMPERSONATOR"
      Permissions:
        - "admin.impersonation"
        - "impersonation"
    - Role: "IAM_END_USER_IMPERSONATOR"
      Permissions:
        - "impersonation"
    - Role: "ORG_OWNER"
      Permissions:
        - "org.read"
        - "org.global.read"
        - "org.write"
        - "org.delete"
        - "org.member.read"
        - "org.member.write"
        - "org.member.delete"
        - "org.idp.read"
        - "org.idp.write"
        - "org.idp.delete"
        - "org.action.read"
        - "org.action.write"
        - "org.action.delete"
        - "org.flow.read"
        - "org.flow.write"
        - "org.flow.delete"
        - "org.feature.read"
        - "org.feature.write"
        - "org.feature.delete"
        - "user.read"
        - "user.global.read"
        - "user.write"
        - "user.delete"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
        - "user.credential.write"
        - "user.passkey.write"
        - "user.feature.read"
        - "user.feature.write"
        - "user.feature.delete"
        - "policy.read"
        - "policy.write"
        - "policy.delete"
        - "project.read"
        - "project.create"
        - "project.write"
        - "project.delete"
        - "project.member.read"
        - "project.member.write"
        - "project.member.delete"
        - "project.role.read"
        - "project.role.write"
        - "project.role.delete"
        - "project.app.read"
        - "project.app.write"
        - "project.grant.read"
        - "project.grant.write"
        - "project.grant.delete"
        - "project.grant.member.read"
        - "project.grant.member.write"
        - "project.grant.member.delete"
        - "session.delete"
    - Role: "ORG_USER_MANAGER"
      Permissions:
        - "org.read"
        - "user.read"
        - "user.global.read"
        - "user.write"
        - "user.delete"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
        - "user.feature.read"
        - "user.feature.write"
        - "user.feature.delete"
        - "policy.read"
        - "project.read"
        - "project.role.read"
        - "session.delete"
    - Role: "ORG_OWNER_VIEWER"
      Permissions:
        - "org.read"
        - "org.member.read"
        - "org.idp.read"
        - "org.action.read"
        - "org.flow.read"
        - "org.feature.read"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.membership.read"
        - "user.feature.read"
        - "policy.read"
        - "project.read"
        - "project.member.read"
        - "project.role.read"
        - "project.app.read"
        - "project.grant.read"
        - "project.grant.member.read"
        - "project.grant.user.grant.read"
    - Role: "ORG_SETTINGS_MANAGER"
      Permissions:
        - "org.read"
        - "org.write"
        - "org.member.read"
        - "org.idp.read"
        - "org.idp.write"
        - "org.idp.delete"
        - "org.feature.read"
        - "org.feature.write"
        - "org.feature.delete"
        - "policy.read"
        - "policy.write"
        - "policy.delete"
    - Role: "ORG_USER_PERMISSION_EDITOR"
      Permissions:
        - "org.read"
        - "org.member.read"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "policy.read"
        - "project.read"
        - "project.member.read"
        - "project.role.read"
        - "project.app.read"
        - "project.grant.read"
        - "project.grant.member.read"
    - Role: "ORG_PROJECT_PERMISSION_EDITOR"
      Permissions:
        - "org.read"
        - "org.member.read"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "policy.read"
        - "project.read"
        - "project.member.read"
        - "project.role.read"
        - "project.app.read"
        - "project.grant.read"
        - "project.grant.write"
        - "project.grant.delete"
        - "project.grant.member.read"
    - Role: "ORG_PROJECT_CREATOR"
      Permissions:
        - "user.global.read"
        - "policy.read"
        - "project.read:self"
        - "project.create"
    - Role: "ORG_ADMIN_IMPERSONATOR"
      Permissions:
        - "admin.impersonation"
        - "impersonation"
    - Role: "ORG_END_USER_IMPERSONATOR"
      Permissions:
        - "impersonation"
    - Role: "PROJECT_OWNER"
      Permissions:
        - "org.global.read"
        - "policy.read"
        - "project.read"
        - "project.write"
        - "project.delete"
        - "project.member.read"
        - "project.member.write"
        - "project.member.delete"
        - "project.role.read"
        - "project.role.write"
        - "project.role.delete"
        - "project.app.read"
        - "project.app.write"
        - "project.app.delete"
        - "project.grant.read"
        - "project.grant.write"
        - "project.grant.delete"
        - "project.grant.member.read"
        - "project.grant.member.write"
        - "project.grant.member.delete"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
    - Role: "PROJECT_OWNER_VIEWER"
      Permissions:
        - "policy.read"
        - "project.read"
        - "project.member.read"
        - "project.role.read"
        - "project.app.read"
        - "project.grant.read"
        - "project.grant.member.read"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.membership.read"
    - Role: "SELF_MANAGEMENT_GLOBAL"
      Permissions:
        - "org.create"
        - "policy.read"
        - "user.self.delete"
    - Role: "ORG_USER_SELF_MANAGER"
      Permissions:
        - "policy.read"
        - "user.self.delete"
    - Role: "PROJECT_OWNER_GLOBAL"
      Permissions:
        - "org.global.read"
        - "policy.read"
        - "project.read"
        - "project.write"
        - "project.delete"
        - "project.member.read"
        - "project.member.write"
        - "project.member.delete"
        - "project.role.read"
        - "project.role.write"
        - "project.role.delete"
        - "project.app.read"
        - "project.app.write"
        - "project.app.delete"
        - "user.global.read"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
    - Role: "PROJECT_OWNER_VIEWER_GLOBAL"
      Permissions:
        - "policy.read"
        - "project.read"
        - "project.member.read"
        - "project.role.read"
        - "project.app.read"
        - "project.grant.read"
        - "project.grant.member.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.membership.read"
    - Role: "PROJECT_GRANT_OWNER"
      Permissions:
        - "policy.read"
        - "org.global.read"
        - "project.read"
        - "project.grant.read"
        - "project.grant.member.read"
        - "project.grant.member.write"
        - "project.grant.member.delete"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.grant.write"
        - "user.grant.delete"
        - "user.membership.read"
    - Role: "PROJECT_GRANT_OWNER_VIEWER"
      Permissions:
        - "policy.read"
        - "project.read"
        - "project.grant.read"
        - "project.grant.member.read"
        - "user.read"
        - "user.global.read"
        - "user.grant.read"
        - "user.membership.read"

# If a new projection is introduced it will be prefilled during the setup process (if enabled)
# This can prevent serving outdated data after a version upgrade, but might require a longer setup / upgrade process:
# https://zitadel.com/docs/self-hosting/manage/updating_scaling
InitProjections:
  Enabled: true # ZITADEL_INITPROJECTIONS_ENABLED
  RetryFailedAfter: 100ms # ZITADEL_INITPROJECTIONS_RETRYFAILEDAFTER
  MaxFailureCount: 2 # ZITADEL_INITPROJECTIONS_MAXFAILURECOUNT
  BulkLimit: 1000 # ZITADEL_INITPROJECTIONS_BULKLIMIT
```
```
zitadel start-from-init   --config defaults.yaml  --masterkey "MasterkeyNeedsToHave32Characters"  --tlsMode external
```
### Zitadel Service
```
vi /etc/systemd/system/zitadel.service
```
```
[Unit]
Description=zitadel

[Service]
Type=simple
User=root
Group=root
EnvironmentFile=-/usr/local/bin/zitadel
ExecStart=/usr/local/bin/zitadel  start  --masterkey "MasterkeyNeedsToHave32Characters"  --tlsMode external
WorkingDirectory=/usr/local/bin/
Restart=always
Nice=19
LimitNOFILE=16384
# When stopping, how long to wait before giving up and sending SIGKILL?
# Keep in mind that SIGKILL on a process can cause data loss.
TimeoutStopSec=60

[Install]
WantedBy=multi-user.target
```
```
systemctl daemon-reload
```
```
systemctl start zitadel
```

### No Isssue,  proceed to  Mirror from CockroachDB to Postgresql
	 
### Postgres Install
```
sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```
```
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
```
```
apt  update
```
```
apt -y install postgresql
```

## Create Zitadel Database
```
sudo -u postgres psql
```
```
CREATE ROLE zitadel LOGIN;
```
```
CREATE DATABASE zitadel;
```
```
GRANT CONNECT, CREATE ON DATABASE zitadel TO zitadel;
```

### Mirror  Configuration file
```
vi config.yaml
```
Add the following to config.yaml file.
```
Source:
  cockroach:
    Host: localhost 
    Port: 26257 
    Database: zitadel 
    MaxOpenConns: 6 
    MaxIdleConns: 6 
    EventPushConnRatio: 0.33 
    ProjectionSpoolerConnRatio: 0.33 
    MaxConnLifetime: 30m 
    MaxConnIdleTime: 5m 
    Options: "" 
    User:
      Username: zitadel 
      Password: "" 
      SSL:
        Mode: disable 
        RootCert: "" 
        Cert: "" 
        Key: "" 
  postgres:
    Host: localhost 
    Port: 5432  
    Database: zitadel  
    MaxOpenConns: 10  
    MaxIdleConns: 10 
    MaxConnLifetime: 1h 
    MaxConnIdleTime: 5m 
    Options: 
    User:
      Username: zitadel  
      Password: zitadel 
      SSL:
        Mode: disable 
        RootCert: 
        Cert: 
        Key: 
EventBulkSize: 10000 
Projections:  
  ConcurrentInstances: 7 
  EventBulkLimit: 1000 

Auth:
  Spooler:
    BulkLimit: 1000 

Admin:
  Spooler:
    BulkLimit: 10 
Log:
  Level: info
```

From the offical documentations

```
zitadel init --config /path/to/your/new/config.yaml
zitadel setup --for-mirror --config /path/to/your/new/config.yaml # make sure to set --tlsMode and masterkey analog to your current deployment
zitadel mirror --system --config /path/to/your/mirror/config.yaml # make sure to set --tlsMode and masterkey analog to your current deployment
```

### First Command executed no issues.

```
root@zitadel003:/usr/local/bin# zitadel init --config /usr/local/bin/config.yaml
INFO[0000] initialization started                        caller="/home/runner/work/zitadel/zitadel/cmd/initialise/init.go:75"
INFO[0000] verify user                                   caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_user.go:39" username=zitadel
INFO[0000] verify database                               caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_database.go:39" database=zitadel
INFO[0000] verify grant                                  caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_grant.go:34" database=zitadel user=zitadel
INFO[0000] verify settings                               caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_settings.go:40" database=zitadel user=zitadel
INFO[0000] verify zitadel                                caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:78" database=zitadel
INFO[0000] verify system                                 caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:39"
INFO[0000] verify encryption keys                        caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:44"
INFO[0000] verify projections                            caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:49"
INFO[0000] verify eventstore                             caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:54"
INFO[0000] verify events tables                          caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:59"
INFO[0000] verify system sequence                        caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:64"
INFO[0000] verify unique constraints                     caller="/home/runner/work/zitadel/zitadel/cmd/initialise/verify_zitadel.go:69"
root@zitadel003:/usr/local/bin#
```

### Second commnad  executeed no issues

 ```
root@zitadel003:/usr/local/bin# zitadel setup --for-mirror --config /usr/local/bin/config.yaml  --masterkey "MasterkeyNeedsToHave32Characters" --tlsMode external
INFO[0000] setup started                                 caller="/home/runner/work/zitadel/zitadel/cmd/setup/setup.go:99"
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=14_events_push
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=01_tables
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=02_assets
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=03_default_instance
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=05_last_failed
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=06_resource_owner_columns
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=07_logstore
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=08_auth_token_indexes
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=12_auth_users_otp_columns
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=13_fix_quota_constraints
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=15_current_projection_state
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=16_unique_constraint_lower
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=17_add_offset_col_to_current_states
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=19_add_current_sequences_index
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=20_add_by_user_index_on_session
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=22_active_instance_events_index
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=23_correct_global_unique_constraints
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=24_add_actor_col_to_auth_tokens
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=26_auth_users3
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=config_change
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=projection_tables
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=18_add_lower_fields_to_login_names
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=21_add_block_field_to_limits
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=25_user12_add_lower_fields_to_verified_email
INFO[0000] verify migration                              caller="/home/runner/work/zitadel/zitadel/internal/migration/migration.go:43" name=26_idp_templates6_add_saml_name_id_format
root@zitadel003:/usr/local/bin#
```
###  Execute this mirror command I had issues.

```
zitadel mirror --system --replace --config /usr/local/bin/config.yaml  --masterkey "MasterkeyNeedsToHave32Characters" --tlsMode external
```

Recieve the following error.

```
root@zitadel003:/usr/local/bin# zitadel mirror --system  --replace   --config /usr/local/bin/config.yaml  --masterkey "MasterkeyNeedsToHave32Characters" --tlsMode external
INFO[0000] assets migrated                               caller="/home/runner/work/zitadel/zitadel/cmd/mirror/system.go:92" count=1 took=35.159277ms
INFO[0000] encryption keys migrated                      caller="/home/runner/work/zitadel/zitadel/cmd/mirror/system.go:138" count=10 took=197.355795ms
INFO[0000] auth requests migrated                        caller="/home/runner/work/zitadel/zitadel/cmd/mirror/auth.go:90" count=2 took=17.140438ms
INFO[0000] start event migration                         caller="/home/runner/work/zitadel/zitadel/cmd/mirror/event_store.go:97" from=0 to=1.7207433088659942e+18
ERRO[0000] unable to mirror events                       caller="/home/runner/work/zitadel/zitadel/cmd/mirror/event_store.go:191" error="ID=MIGRA-DTHi7 Message=unable to copy events into destination Parent=(ERROR: duplicate key value violates unique constraint \"events2_pkey\" (SQLSTATE 23505))"
FATA[0000] unable to write failed event                  caller="/home/runner/work/zitadel/zitadel/cmd/mirror/event_store.go:193" error="ID=POSTG-KOM6E Message=Errors.Internal.Eventstore.SequenceNotMatched"
root@zitadel003:/usr/local/bin# 
```



