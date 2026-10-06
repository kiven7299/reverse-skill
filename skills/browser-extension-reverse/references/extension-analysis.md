# Extension analysis notes

| Field | Risk signal |
|------|----------|
| host_permissions `<all_urls>` | Read/write any site |
| webRequestBlocking | MitM-style rewrite |
| nativeMessaging | Browser to host |
| externally_connectable | Page drives the extension |

MV3: watch service_worker lifecycle and declarativeNetRequest.
