# Token-based HTTP Resource Access Control using Access Policy Manager’s HTTP Connector

## Overview

Imagine a scenario where you are responsible for providing millions of proprietary hardware devices with access to application services using intelligence in the core of the network. This could be a VOD service, a voice/video conferencing resource or an IP camera. How can we incorporate application-level based access control to these resources with little-to-no intervention on the client and server?

We can use APM to provide per-client and per-request policies and incorporate an authentication mechanism that is both transparent and out-of-band of the session. In this example, we’ll show how to use HTTP Connector to intercept an HTTP request, call an authentication API server and pass through a client identifier to make a decision if the connection should be allowed.  

In this case, the CPE registers and authenticates to a central auth service during boot, obtains and then passes along a unique identifier when requesting internal resources. It does so utilizing protocol headers. This makes it easy to not only identify the client, but to verify the identity back to the authentication server.  

## Configuration

### LTM

This section contains the node, pool and virtual server configurations. Our application lives at 10.1.20.6 on port 80.

Note that you could easily attach an SSL profile to the client and/or server side. It would be necessary to terminate SSL at the virtual server if it is in use between the client and server resources, as we need to extract the token from the HTTP header.

Node

```console
ltm node 10.1.20.6 {
    address 10.1.20.6
}
```

Pool

```console
ltm pool pool_node {
    members {
        10.1.20.6:80 {
            address 10.1.20.6
            session monitor-enabled
            state up
        }
    }
    monitor gateway_icmp
}
```

Virtual Server

```console
ltm virtual api-via-token {
    destination 10.1.10.100:http
    ip-protocol tcp
    mask 255.255.255.255
    per-flow-request-access-policy api-via-token
    pool pool_node
    profiles {
        http { }
        rba { }
        tcp { }
        test-ltm-apm { }
        websso { }
    }
    serverssl-use-sni disabled
    source 0.0.0.0/0
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
}
```

### APM

```console
apm aaa http-connector-request api-via-token {
    app-service none
    auth none
    method GET
    partition Common
    password none
    request-body none
    request-headers "user-id: %{perflow.username}"
    response-action parse
    response-headers none
    token none
    transport www-example-com-resolver
    url http://10.1.10.6:3000
    username none
}
apm aaa http-connector-transport www-example-com-resolver {
    app-service none
    dns-resolver www-example-com-resolver
    max-response-size 32768
    partition Common
    ssl-profile none
    target-vs none
    timeout 5
}
apm policy access-policy api-via-token {
    app-service none
    caption none
    default-ending api-via-token_end_reject
    items {
        api-via-token_end_allow {
            priority 0
        }
        api-via-token_end_reject {
            priority 0
        }
        api-via-token_ent {
            priority 0
        }
        api-via-token_mac_token_api_call {
            priority 0
        }
    }
    macros { /Common/token_api_call }
    max-macro-loop-count 1
    oneshot-macro false
    partition Common
    per-req-policy-properties {
        api-via-token {
            app-service none
            incomplete-action deny
        }
    }
    start-item api-via-token_ent
    subroutine-properties none
    type per-rq-policy
}
apm policy access-policy test-ltm-apm {
    app-service none
    caption none
    default-ending test-ltm-apm_end_deny
    items {
        test-ltm-apm_end_allow {
            priority 0
        }
        test-ltm-apm_end_deny {
            priority 0
        }
        test-ltm-apm_ent {
            priority 0
        }
    }
    macros none
    max-macro-loop-count 1
    oneshot-macro false
    partition Common
    per-req-policy-properties none
    start-item test-ltm-apm_ent
    subroutine-properties none
    type access-policy
}
apm policy access-policy token_api_call {
    app-service none
    caption token_api_call
    default-ending token_api_call_ter_deny
    items {
        token_api_call_act_http_connector {
            priority 0
        }
        token_api_call_ent_in {
            priority 0
        }
        token_api_call_ter_deny {
            priority 5
        }
        token_api_call_ter_out {
            priority 4
        }
    }
    macros none
    max-macro-loop-count 1
    oneshot-macro false
    partition Common
    per-req-policy-properties none
    start-item token_api_call_ent_in
    subroutine-properties {
        token_api_call {
            app-service none
            gating-criteria none
            inactivity-timeout 300
            max-subsession-lifetime 900
            subroutine-timeout 120
        }
    }
    type subroutine
}
apm policy agent ending-allow api-via-token_end_allow_ag {
    app-service none
    partition Common
}
apm policy agent ending-allow test-ltm-apm_end_allow_ag {
    app-service none
    partition Common
}
apm policy agent ending-deny test-ltm-apm_end_deny_ag {
    app-service none
    customization-group test-ltm-apm_end_deny_ag
    partition Common
}
apm policy agent ending-reject api-via-token_end_reject_ag {
    app-service none
    partition Common
    response none
}
apm policy agent http-connector token_api_call_act_http_connector_ag {
    app-service none
    partition Common
    request-name api-via-token
}
apm policy customization-group api-via-token_eps {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:01:26
    created-by admin
    images none
    last-update-time 2021-08-10:19:01:26
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type eps
    updated-by admin
}
apm policy customization-group api-via-token_errormap {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:01:26
    created-by admin
    images none
    last-update-time 2021-08-10:19:01:26
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type errormap
    updated-by admin
}
apm policy customization-group api-via-token_framework_installation {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:01:26
    created-by admin
    images none
    last-update-time 2021-08-10:19:01:26
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type framework-installation
    updated-by admin
}
apm policy customization-group api-via-token_general_ui {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:01:26
    created-by admin
    images none
    last-update-time 2021-08-10:19:01:26
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type general-ui
    updated-by admin
}
apm policy customization-group api-via-token_logout {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:01:26
    created-by admin
    images none
    last-update-time 2021-08-10:19:01:26
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type logout
    updated-by admin
}
apm policy customization-group test-ltm-apm_end_deny_ag {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:34:18
    created-by admin
    images none
    last-update-time 2021-08-10:19:34:18
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type logout
    updated-by admin
}
apm policy customization-group test-ltm-apm_eps {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:34:18
    created-by admin
    images none
    last-update-time 2021-08-10:19:34:18
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type eps
    updated-by admin
}
apm policy customization-group test-ltm-apm_errormap {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:34:18
    created-by admin
    images none
    last-update-time 2021-08-10:19:34:18
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type errormap
    updated-by admin
}
apm policy customization-group test-ltm-apm_framework_installation {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:34:18
    created-by admin
    images none
    last-update-time 2021-08-10:19:34:18
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type framework-installation
    updated-by admin
}
apm policy customization-group test-ltm-apm_general_ui {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:34:18
    created-by admin
    images none
    last-update-time 2021-08-10:19:34:18
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type general-ui
    updated-by admin
}
apm policy customization-group test-ltm-apm_logout {
    app-service none
    checksum SHA1:62:fd61541c1097d460e42c50904684def2794ba70d
    create-time 2021-08-10:19:34:18
    created-by admin
    images none
    last-update-time 2021-08-10:19:34:18
    local-path none
    mode 33188
    partition Common
    revision 1
    size 62
    source modern
    templates none
    type logout
    updated-by admin
}
apm policy customization-group-set api-via-token {
    accept-languages { en }
    access-policy api-via-token
    app-service none
    cache-generation 1
    customization-group api-via-token_logout
    customization-key d989d7329a09e2271d016ca72f18b978
    default-language en
    eps-group api-via-token_eps
    errormap-group api-via-token_errormap
    framework-installation-group api-via-token_framework_installation
    general-ui-group api-via-token_general_ui
    partition Common
}
apm policy customization-source modern {
    app-service none
    partition Common
}
apm policy customization-source standard {
    app-service none
    partition Common
}
apm policy policy-item api-via-token_end_allow {
    agents {
        api-via-token_end_allow_ag {
            type ending-allow
        }
    }
    app-service none
    caption Allow
    color 1
    item-type ending
    loop false
    macro none
    partition Common
    rules none
}
apm policy policy-item api-via-token_end_reject {
    agents {
        api-via-token_end_reject_ag {
            type ending-reject
        }
    }
    app-service none
    caption Reject
    color 2
    item-type ending
    loop false
    macro none
    partition Common
    rules none
}
apm policy policy-item api-via-token_ent {
    agents none
    app-service none
    caption Start
    color 1
    item-type entry
    loop false
    macro none
    partition Common
    rules {
        {
            caption fallback
            expression none
            next-item api-via-token_mac_token_api_call
        }
    }
}
apm policy policy-item api-via-token_mac_token_api_call {
    agents none
    app-service none
    caption token_api_call
    color 1
    item-type macro-call
    loop false
    macro token_api_call
    partition Common
    rules {
        {
            caption Permit
            expression none
            next-item api-via-token_end_allow
        }
        {
            caption Deny
            expression none
            next-item api-via-token_end_reject
        }
    }
}
apm policy policy-item test-ltm-apm_end_allow {
    agents {
        test-ltm-apm_end_allow_ag {
            type ending-allow
        }
    }
    app-service none
    caption Allow
    color 1
    item-type ending
    loop false
    macro none
    partition Common
    rules none
}
apm policy policy-item test-ltm-apm_end_deny {
    agents {
        test-ltm-apm_end_deny_ag {
            type ending-deny
        }
    }
    app-service none
    caption Deny
    color 2
    item-type ending
    loop false
    macro none
    partition Common
    rules none
}
apm policy policy-item test-ltm-apm_ent {
    agents none
    app-service none
    caption Start
    color 1
    item-type entry
    loop false
    macro none
    partition Common
    rules {
        {
            caption fallback
            expression none
            next-item test-ltm-apm_end_allow
        }
    }
}
apm policy policy-item token_api_call_act_http_connector {
    agents {
        token_api_call_act_http_connector_ag {
            type http-connector
        }
    }
    app-service none
    caption "HTTP Connector"
    color 1
    item-type action
    loop false
    macro none
    partition Common
    rules {
        {
            caption Successful
            expression "expr { [mcget {subsession.http_connector.status}] == 200 && [mcget {subsession.http_connector.error}] == 0}"
            next-item token_api_call_ter_out
        }
        {
            caption fallback
            expression none
            next-item token_api_call_ter_deny
        }
    }
}
apm policy policy-item token_api_call_ent_in {
    agents none
    app-service none
    caption In
    color 1
    item-type entry
    loop false
    macro none
    partition Common
    rules {
        {
            caption fallback
            expression none
            next-item token_api_call_act_http_connector
        }
    }
}
apm policy policy-item token_api_call_ter_deny {
    agents none
    app-service none
    caption Deny
    color 2
    item-type terminal-out
    loop false
    macro none
    partition Common
    rules none
}
apm policy policy-item token_api_call_ter_out {
    agents none
    app-service none
    caption Permit
    color 1
    item-type terminal-out
    loop false
    macro none
    partition Common
    rules none
}
apm profile access test-ltm-apm {
    accept-languages { en }
    access-policy test-ltm-apm
    access-policy-timeout 300
    app-service none
    customization-group test-ltm-apm_logout
    customization-key 1eade7c6651ee2e08facda32b30f8019
    default-language en
    defaults-from access
    domain-cookie none
    domain-groups none
    domain-mode single-domain
    enforce-policy true
    eps-group test-ltm-apm_eps
    errormap-group test-ltm-apm_errormap
    exchange-profile none
    framework-installation-group test-ltm-apm_framework_installation
    general-ui-group test-ltm-apm_general_ui
    generation 6
    generation-action noop
    httponly-cookie false
    inactivity-timeout 900
    log-settings {
        default-log-setting
    }
    logout-uri-include none
    logout-uri-timeout 5
    max-concurrent-sessions 0
    max-concurrent-users 0
    max-failure-delay 5
    max-in-progress-sessions 128
    max-session-timeout 604800
    min-failure-delay 2
    modified-since-last-policy-sync true
    named-scope none
    ntlm-auth-name none
    oauth-profile none
    partition Common
    persistent-cookie false
    primary-auth-service none
    restrict-to-single-client-ip false
    samesite-cookie false
    samesite-cookie-attr-value strict
    sandboxes none
    scope profile
    secure-cookie true
    sso-name none
    type ltm-apm
    use-http-503-on-error true
    user-identity-method ip-address
    webtop-redirect-on-root-uri false
}
```

## Usage

Outline how the user can use your project and the various features the project offers.

## Development

Outline any requirements to setup a development environment if someone would like to contribute.  You may also link to another file for this information.

## Support

For support, please open a GitHub issue.  Note, the code in this repository is community supported and is not supported by F5 Networks.  For a complete list of supported projects please reference [SUPPORT.md](SUPPORT.md).

## Community Code of Conduct

Please refer to the [F5 DevCentral Community Code of Conduct](code_of_conduct.md).

## License

[Apache License 2.0](LICENSE)

## Copyright

Copyright 2014-2020 F5 Networks Inc.

### F5 Networks Contributor License Agreement

Before you start contributing to any project sponsored by F5 Networks, Inc. (F5) on GitHub, you will need to sign a Contributor License Agreement (CLA).

If you are signing as an individual, we recommend that you talk to your employer (if applicable) before signing the CLA since some employment agreements may have restrictions on your contributions to other projects.
Otherwise by submitting a CLA you represent that you are legally entitled to grant the licenses recited therein.

If your employer has rights to intellectual property that you create, such as your contributions, you represent that you have received permission to make contributions on behalf of that employer, that your employer has waived such rights for your contributions, or that your employer has executed a separate CLA with F5.

If you are signing on behalf of a company, you represent that you are legally entitled to grant the license recited therein.
You represent further that each employee of the entity that submits contributions is authorized to submit such contributions on behalf of the entity pursuant to the CLA.
