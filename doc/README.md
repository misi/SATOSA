# SATOSA
This document describes how to install and configure the SATOSA proxy.


## Installation
1. Download the SATOSA proxy project as a [compressed archive ](https://github.com/its-dirg/SATOSA/releases)
   and unpack it to `<satosa_path>`.

1. Install the Python code and its requirements:

   ```bash  
   pip install <satosa_path> -r <satosa_path>/requirements.txt
   ```

## Configuration
All default configuration files, as well as an example WSGI application for the proxy, can be found
in the [example directory](../example).


### SATOSA proxy configuration: `proxy_conf.yaml.example`
| Parameter name | Data type | Example value | Description |
| -------------- | --------- | ------------- | ----------- |
| `BASE` | string | `https://proxy.example.com` | base url of the proxy |
| `SESSION_OPTS` | dict | `{session.type: memory, session.cookie_expires: Yes, session.auto: Yes}` | configuration options for [Beaker Session Middleware](http://beaker.readthedocs.org/en/latest/configuration.html)
| `COOKIE_STATE_NAME` | string | `vopaas_state` | name of cooke VOPaaS uses for preserving state between requests |
| `STATE_ENCRYPTION_KEY` | string | `52fddd3528a44157` | key used for encrypting the state cookie, will be overriden by the environment variable `SATOSA_STATE_ENCRYPTION_KEY` if it is set |
| `INTERNAL_ATTRIBUTES` | string | `example/internal_attributes.yaml` | path to attribute mapping
| `PLUGIN_PATH` | string[] | `[example/plugins/backends, example/plugins/frontends]` | list of directory paths containing any front-/backend plugins |
| `BACKEND_MODULES` | string[] | `[oidc_backend, saml2_backend]` | list of plugin names to load from the directories in `PLUGIN_PATH` |
| `FRONTEND_MODULES` | string[] | `[saml2_frontend]` | list of plugin names to load from the directories in `PLUGIN_PATH` |
| `USER_ID_HASH_SALT` | string | `61a89d2db0b9e1e2` | salt used when creating the persistent user identifier, will be overriden by the environment variable `SATOSA_USER_ID_HASH_SALT` if it is set |
| `CONSENT` | dict | see configuration of [Additional Services](#additional-services) | optional configuration of consent service |
| `ACCOUNT_LINKING` | dict | see configuration of [Additional Services](#additional-services) | optional configuration of account linking service |


#### Additional services
| Parameter name | Data type | Example value | Description |
| -------------- | --------- | ------------- | ----------- |
| `enable` | bool | `Yes` | whether the service should be used |
| `rest_uri` | string | `https://localhost` | url to the REST endpoint of the service |
| `redirect` | string | `https://localhost/redirect` | url to the endpoint where the user should be redirected for necessary interaction |
| `endpoint` | string | `handle_consent` | name of the endpoint in VOPaas where the response from the service is received |
| `sign_key`| string | `pki/consent.key` | path to key used for signing the requests to the service |
| `verify_ssl` | bool | `No` | whether the HTTPS certificate of the service should be verified when doing requests to it |

If using the [CMService](https://github.com/its-dirg/CMservice) for consent management and the [ALService](https://github.com/its-dirg/ALservice) for account linking, the `redirect` parameter should be `https://<host>/consent` and `https://<host>/approve` in the respective configuration entry.


### Attribute mapping configuration: `internal_attributes.yaml`

#### `attributes`
The values directly under the `attributes` key are the internal attribute names.
Every internal attribute has a map of profiles, which in turn has a list of
external attributes names which should be mapped to the internal attributes.

If multiple external attributes are specified under a profile, the proxy will
store all attribute values from the external attributes as a list in the
internal attribute.

Sometimes the external attributes are nested/complex structures. One example is
the [address claim in OpenID connect](http://openid.net/specs/openid-connect-core-1_0.html#AddressClaim)
which consists of multiple sub-fields, e.g.:
```json
"address": {
  "formatted": "100 Universal City Plaza, Hollywood CA 91608, USA",
  "street_address": "100 Universal City Plaza",
  "locality": "Hollywood",
  "region": "CA",
  "postal_code": "91608",
  "country": "USA",
}
```

In this case the proxy accepts a dot-separated string denoting which external
attribute to use, e.g. `address.formatted`.

**Example**
```yaml
attributes:
  mail:
    openid: [email]
    saml: [mail, emailAdress, email]
  address:
    openid: [address.formatted]
    saml: [postaladdress]
```

This example defines two attributes internal to the proxy, named `mail` and `address` accessible to any plugins (e.g. front- and backends) in the proxy and their mapping
in two different "profiles", named `openid` and `saml` respectively.

The mapping between received attributes (proxy backend) <-> internal <-> returned attributes (proxy frontend) is defined as:

* Any plugin using the `openid` profile will use the attribute value from
  `email` delivered from the backing provider as the value for `mail`.
* Any plugin using the `saml` profile will use the attribute value from `mail`,
  `emailAdress` and `email` depending on which attributes are delivered by the
  backing provider as the value for `mail`.
* Any plugin using the `openid` profile will use the attribute value under the
  key `formatted` in the `address` attribute delivered by the backing provider.


#### `user_id_from_attr`
The user identifier generated by the backend module can be overridden by
specifying a list of internal attribute names under the `user_id_from_attr` key.
The attribute values of the attributes specified in this list will be
concatenated and hashed to be used as the user identifier.


#### `user_id_to_attr`
To store the user identifier in a specific internal attribute, the internal
attribute name can be specified in `user_id_to_attr`.
When the [ALService](https://github.com/its-dirg/ALservice) is used the
`user_id_to_attr` should be used, since that account linking service will
overwrite the user identifier generated by the proxy.


#### `hash`
The proxy can hash any attribute value (e.g., for obfuscation) before passing
it on to the client. The `hash` key should contain a list of all attribute names
for which the corresponding attribute value should be hashed before being
returned to the client.


## Plugins
The protocol specific communication is handled by different plugins, divided
into frontends (receiving requests from clients) and backends (sending requests
to backing identity providers).

### SAML2 plugins

Common configuration parameters:

| Parameter name | Data type | Example value | Description |
| -------------- | --------- | ------------- | ----------- |
| `organization` | dict | `{display_name: Example Identities, name: Example Identities Organization, url: https://www.example.com}` | information about the organization, will be published in the SAML metadata |
| `contact_person` | dict[] | `{contact_type: technical, given_name: Someone Technical, email_address: technical@example.com}` | list of contact information, will be published in the SAML metadata |
| `key_file` | string | `pki/key.pem` | path to private key used for signing(backend)/decrypting(frontend) SAML2 assertions |
| `cert_file` | string | `pki/cert.pem` | path to certificate for the public key associated with the private key in `key_file` |
| `metadata["local"]` | string[] | `[metadata/entity.xml]` | list of paths to metadata for all service providers (frontend)/identity providers (backend) communicating with the proxy |

For more detailed information on how you could customize the SAML entities configuration please visit: 
https://dirg.org.umu.se/static/pysaml2/howto/config.html

#### Metadata
The metadata could be loaded in multiple ways in the table above it's loaded from a static 
file by using the key "local". It's also possible to load read the metadata from a remote URL.
The final way to load metadata is by using a discovery server, in order to achieve this enter an URL
in the "disco_srv".

**Examples:**

Metadata from local file:

    "metadata": 
        local: [idp.xml]

Metadata from remote URL:

    "metadata": {
        "remote": 
            -url:https://kalmar2.org/simplesaml/module.php/aggregator/?id=kalmarcentral2&set=saml2,
            -cert:null
    }

Metadata from discovery server:

    disco_srv: http://disco.example.com
    

#### Frontend
The SAML2 frontend acts as an SAML Identity Provider (IdP), accepting
authentication requests from SAML Service Providers (SP). The default
configuration file can be found [here](../example/plugins/frontends/saml2_frontend.yaml.example).


#### Backend
The SAML2 backend acts as an SAML Service Provider (SP), making authentication
requests to SAML Identity Providers (IdP). The default configuration file can be
found [here](../example/plugins/backends/saml2_backend.yaml.example).


### OpenID Connect plugins

#### Backend
The OpenID Connect backend acts as an OpenID Connect Relying Party (RP), making
authentication requests to OpenID Connect Provider (OP). The default
configuration file can be found [here](../example/plugins/backends/openid_backend.yaml.example).

Only the `op_url` must be configured to specify the OP issuer url.

The example configuration assumes the OP supports [discovery](http://openid.net/specs/openid-connect-discovery-1_0.html)
and [dynamic client registration](https://openid.net/specs/openid-connect-registration-1_0.html).
When using an OP that only supports statically registered clients, see the
[default configuration for using Google as the OP](../example/plugins/backends/google_backend.yaml.example).


### Social login plugins
The social login plugins can be used as backends for the proxy, allowing the
proxy to act as a client to the social login services.

#### Google
The default configuration file can be
found [here](../example/plugins/backends/google_backend.yaml.example).

The only parameters necessary to configure is the credentials,
the `client_id` and `client_secret`, issued by Google. See [OAuth 2.0 credentials](https://developers.google.com/identity/protocols/OpenIDConnect#getcredentials)
for information on how to obtain them.

The `redirect_uri` of the SATOSA proxy must be registered with Google. The
redirect URI to register with Google is "<base_url>/google", where `<base_url>`
is the base url of the proxy as specified in the `BASE` configuration parameter
in `proxy_conf.yaml`, e.g. "https://proxy.example.com/google".

A list of all claims possibly released by Google can be found [here](https://developers.google.com/identity/protocols/OpenIDConnect#obtainuserinfo),
which should be used when configuring the attribute mapping (see above).


#### Facebook
The default configuration file can be
found [here](../example/plugins/backends/fb_backend.yaml.example).

The only parameters necessary to configure is the credentials,
the "App ID" (`client_id`) and "App Secret" (`client_secret`), issued by Facebook.
See the [registration instructions](https://developers.facebook.com/docs/apps/register)
for information on how to obtain them.


## SAML metadata

The SAML metadata of the proxy is generated based on the `proxy_conf.yaml`
(which defines all front-/backend plugins) using the `make_saml_metadata.py`
(installed globally by SATOSA installation).

### Generate backend metadata
The command
```bash
make_saml_metadata.py proxy_conf.yaml
```
will generate separate metadata files for all SAML2 backend modules specified in
`proxy_conf.yaml`.

Detailed usage instructions can be viewed by running `make_saml_metadata.py -h`.

## Running the proxy application
Start the proxy server with the following command:
```bash
gunicorn -b<socket address> satosa.wsgi:app --keyfile=<https key> --certfile=<https cert>
```
where
* `socket address` is the socket address that `gunicorn` should bind to for incoming requests, e.g. `0.0.0.0:8080`
* `https key` is the path to the private key to use for HTTPS, e.g. `pki/key.pem`
* `https cert` is the path to the certificate to use for HTTPS, e.g. `pki/cert.pem`

This will use the `proxy_conf.yaml` file in the working directory. If the `proxy_conf.yaml` is
located somewhere else, use the environment variable `SATOSA_CONFIG` to specify the path, e.g.:

```bash
set SATOSA_CONFIG=/home/user/proxy_conf.yaml
```