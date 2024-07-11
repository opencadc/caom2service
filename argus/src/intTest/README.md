# argus integration tests

The tests do a registry lookup of resourceID `ivo://opencadc.org/argus`.

Most of the tests are anonymous, but a few require authentication with an X509 client and are only feasible 
for a CADC staff member to run. Required certificates:
* `$A/test-certificates/argus-auth.pem` : a user that can send async output to ivo://cadc.nrc.ca/vault
* `$A/test-certificates/argus-noauth.pem` : a valid user that has no permission to see proprietary metadata

The _auth_ identity has to be the (CADC) staff member's personal certificate because the test tries to output
to `vos://cadc.nrc.ca~vault/{user.name}/test/some-file-name`. The _noauth_ identity should be cadcregtest1.

An example of a minimum local configuration required for the `argus` service to run locally consists of the
following containers:
- `haproxy` (image: `cadc-haproxy-dev`) - for https termination
- `postgress` (`cadc-postgresql-dev`) - database container
- `reg` (`reg`) - local registry entries
- `icewind` (`icewind`) - to create the `caom2` schema and tables
- `argus` (`argus`) - the service itself

The configuration for the required services is documented in their GitHub repo. For `arugs` the minimum
configuration that successfully run most of the integration tests consists of the following files in `/config`:
- `argus.properties`:
    ```
    org.opencadc.argus.VosiCapabilitiesTest > testTokenAuth`
    ```
- `cadc-registry.properties`:
    ```
    # local authority map to find the service that provides
    # the local implementation of an API (standardID)
    #
    # configure RegistryClient bootstrap
    ca.nrc.cadc.reg.client.RegistryClient.baseURL = <local reg URL>
    # configure LocalAuthority lookups
    # <standardID> = <resourceID of service that provides the API>
    ivo://ivoa.net/std/GMS#search-1.0 = ivo://cadc.nrc.ca/gms
    ivo://ivoa.net/std/GMS#groups-0.1 = ivo://cadc.nrc.ca/gms

    ivo://ivoa.net/std/GMS#search-0.1 = ivo://cadc.nrc.ca/gms
    ivo://ivoa.net/std/UMS#users-0.1 = ivo://cadc.nrc.ca/gms
    ivo://ivoa.net/std/UMS#login-0.1 = ivo://cadc.nrc.ca/gms

    ivo://ivoa.net/sso#tls-with-password = https://ska-iam.stfc.ac.uk/
    ivo://ivoa.net/sso#OpenID = https://ws-cadc.canfar.net/ac

    ivo://ivoa.net/std/CDP#delegate-1.0 = ivo://cadc.nrc.ca/cred
    ivo://ivoa.net/std/CDP#proxy-1.0 = ivo://cadc.nrc.ca/cred
    ```

- `cadc-tap-tmp.properties` (temporary storage in the container):
    ```
    org.opencadc.tap.tmp.StorageManager = org.opencadc.tap.tmp.TempStorageManager
    org.opencadc.tap.tmp.TempStorageManager.baseURL = https://<local argus URL>/argus/results
    org.opencadc.tap.tmp.TempStorageManager.baseStorageDir = /var/tmp/argus
  ```

- `catalina.properties`:
    ```
    # tomcat-base
    tomcat.connector.secure=true
    tomcat.connector.scheme=https
    tomcat.connector.proxyName=<name of the local host>
    tomcat.connector.proxyPort=443
    # enable support for haproxy SSL termination + pass client cert
    ca.nrc.cadc.auth.PrincipalExtractor.enableClientCertHeader=true
    # force all registry lookups local
    ca.nrc.cadc.reg.client.RegistryClient.host=<name of local host>
    # database connection pools
    # tapadm - assuming pgdev:5432 is where the database is deployed
    org.opencadc.argus.tapadm.maxActive=2
    org.opencadc.argus.tapadm.username=tapadm
    org.opencadc.argus.tapadm.password=pw-tapadm
    org.opencadc.argus.tapadm.url=jdbc:postgresql://pgdev:5432/cadctest
    # async
    org.opencadc.argus.uws.maxActive=1
    org.opencadc.argus.uws.username=tapadm
    org.opencadc.argus.uws.password=pw-tapadm
    org.opencadc.argus.uws.url=jdbc:postgresql://pgdev:5432/cadctest
    
    org.opencadc.argus.query.maxActive=1
    # optional: config for separate query pool
    org.opencadc.argus.query.username=tapuser
    org.opencadc.argus.query.password=pw-tapuser
    org.opencadc.argus.query.url=jdbc:postgresql://pgdev:5432/cadctest
    
    # enable support for haproxy SSL termination + pass client cert
    ca.nrc.cadc.auth.PrincipalExtractor.enableClientCertHeader=true
    
    ca.nrc.cadc.auth.IdentityManager=ca.nrc.cadc.ac.ACIdentityManager
    # or ca.nrc.cadc.auth.IdentityManager=org.opencadc.auth.StandardIdentityManager
  ```

