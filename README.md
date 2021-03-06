# Storage Policy API.

Mike O'Neill, April 2019

Contributors:
* Mike O'Neill <[michael.oneill@baycloud.com](mailto:michael.oneill@baycloud.com)>


## Introduction
It has long been recognised that user agent storage has major privacy implications for privacy,
such as the ease with which mechanisms like HTTP cookies, designed to support session state persistence, 
have been used to enable invisible long term surveillance.

Although cookies were the first browser based storage to be used to track users, 
and are still the simplest and most available technique, 
other storage categories such as the client-side cache,
local and session based named item storage and indexedDB transactional storage have also
been used by client-side script to enable persistent tracking. 
HTTP cookies can be regenerated from items in local storage or IndexedDB, 
and many tracking techniques have been invented that use the browser cache to store or derive unique identifiers
without the the user's ability to reset or control it.

Privacy regulators and legislators have responded to this by introducing 
laws requiring web sites to manage storage in privacy protective ways, 
for example by requiring prior and informed user consent before storage items,
such as cookies or fingerprinting via derived identifiers,
are used.

Unfortunately sites are usually built from a variety of components, such as embedded elements and external scripts,
often supplied by third-party service providers, and these can employ user agent storage outside the control, and sometimes
even the awareness, of site managers.

This can make it difficult for them to make enforceable commitments to the site's users, 
inevitably making the commitments they do make seem inaccurate or even dishonest, 
leading to a loss of trust in sites, brands and publishers.

This document defines a mechanism via which a site can control user agent storage by setting overall limits on its use, 
whether it is associated with the top level site's origin or an embedded "third-party" origin. 
Browsers would implement defaults which would still enable retention of a top-level origin's
state while the user is interacting with it, 
but storage would be cleared some time after the interaction stops. 
Third-party origins would have no access to storage unless their domain names were included in an allow list. 
Any request for storage limits higher than the default, which could only be from a top-level origin,
would result in a user permission prompt.

The defaults would allow applications to store data, such as cookies or local storage,
for short periods such as an hour or so to support required functionality such as "session state" across multiple pages,
but ensure this data is not retained beyond the minimum period required to persist the session.

Web applications can already ensure that all data associated with a controlled origin is removed when no longer required 
by using the Clear Site Data API. 
By including a `Clear-Site-Data` header in a response all or particular categories of stored data can be immediately removed,
including cookies, local storage, indexed database storage, JavaScript execution contexts, the client-side cache etc.

The cookie API already contains an expiry mechanism, i.e. the `expires` or `max-age` attribute,
though this does not stop cookies from being perpetually regenerated. 
This API not only extends a similar mechanism to all origin controlled storage, 
i.e. all the storage types controlled by the `Clear-Site-Data` API,
but also ensures the expiration timer is held off  while the user is interacting with the site. 

Any explicit deletion of storage items, by the user agent, 
script in a browsing context or triggered by a cookie-like expiry, will continue as before i.e. is not affected by changes to the storage policy limits.

## Storage-Policy response header
Sites can limit access to default limits storage by returning a `Storage-Policy` 
response header of the following form:

`Storage-Policy: type all; timeout 1800; allow-names _consent low-entropy-preference; allow-origins example.com `

The header's value contains a **storage policy descriptor**, 
consisting of one or more **storage policy attributes** separated by semicolons `;`.


Where the **storage policy attributes** have the following meaning

| attribute        | default    | description  |
| :-------------: |:-------------:| :----- |
| type  | all | The storage type these limits apply to, either **cache**, **cookies**, **storage**, or **all**. |
| timeout      | 3600 (=1 hour)| Number of seconds of no user activity before all top-level origin storage is cleared. The timer is restarted (unless expired) on detection of user activity. |
| allow-names | 'none' |   space separated list of names of cookies, storage items, or IndexDB databases, that are not to be deleted by this mechanism, or 'none' if all cookies are to be deleted.|
| allow-origins | 'none' |   space separated list of origins that can receive and place cookies, and whose browsing contexts can access other storage, or 'none' if no subresource may access storage. The default is only changed if the associated type attribute is `all` or absent. |

Successive `Storage-Policy` headers can specify separate limits for each storage type, 
or a single header can contain several **storage policy attributes** separated by commas.

If a `Storage-Policy` header contains an attribute specifying a **less restrictive** storage policy than the default, 
i.e. a higher value for the `timeout`  attribute, or an `allow-origins` or `allow-names` list other than `'none'`,
the user agent must alert the user in some way, such as with a permission prompt
similar to the prompt resulting from a call by a nested browsing context to the Storage Access 
API's `document.requestStorageAccess` function. 
If the user declines the prompt user agents must refrain from prompting the user again within at least 24 hours.

If a top level browsing context has a `Storage-Policy` header in its initiating response
then subresources (other than those indicated by an `allow-origins` attribute),
will have any `Set-Cookie` headers in the response ignored, 
any request to the origin will not include any `Cookies` headers, and any resulting
browsing context will have no access to terminal storage. 

Existing storage associated with such a subresource will not be deleted, as it would be for the top-level origin,
but cookies will not be communicated or set, and any associated nested browsing context will not have access to any storage.

When the inactivity timers complete the associated top-level origin storage, other than cookies, items or databases indicated by a `allow-names` list, is deleted.
In the default case, where the `type all` attribute is assumed, this would have the same effect as if the user agent had received a `Clear-Site-Data: "cache", "cookies", "storage", "executionContexts"`
in the response to the next HTTP request to the origin after the user has stopped interacting for the `inactivity-timeout` period.


## JavaScript API
### Determine current duration limits.
Script can execute the following JavaScript function to determine the storage durations associated with the script origin.

```
var storagePolicyObject = navigator.storagePolicy.Get()
```

The returned object would contain the durations currently in effect, the third-party domains
with access to their own origin storage and cookie names not to be deleted on session timeout:
```javascript
{
      [
          {
            type: "cookies",
            maxAge: 3600,
            inactivityTimeout: 1800,
            allowOrigins: 
                            [
                                "example.com",
                                "anotherexmple.com"
                            ],
            allowNames: 
                        [
                            "euconsent"
                        ]
          },
          {
            type: "storage",
            maxAge: 3600,
            inactivityTimeout: 1800,
            allowOrigins: 
                            [
                                "example.com",
                                "anotherexmple.com"
                            ],
          }
      ]
}
```
### Change current duration limits
Script can change the current storage policy duration, 
but only if the browsing context was initialised **without** an associated `Storage-Policy` header.
If there was a `Storage-Policy` header the user agent MUST ignore the call.
```
navigator.storagePolicy.Update(dictionary);
```
where `dictionary` is an object indicating the required storage limits for specified category types:
```javascript
{
      [
          {
            type: "cookies",
            maxAge: 3600,
            inactivityTimeout: 1800,
            allowOrigins: 
                            [
                                "example.com",
                                "anotherexmple.com"
                            ],
            allowNames: 
                        [
                            "euconsent"
                        ]
          },
          {
            type: "storage",
            maxAge: 3600,
            inactivityTimeout: 1800,
            allowOrigins: 
                            [
                                "example.com",
                                "anotherexmple.com"
                            ],
          }
      ]
}

```



## Prior Art
*  The Clear-Site-Data API defines a mechanism to remove data from local storage, 
giving web developers the ability to clear out a user’s local cache of data via the Clear-Site-Data HTTP response header. 
"[Clear Site Data](https://www.w3.org/TR/clear-site-data/)"

*   Mike West has proposed a mechanism which allows HTTP servers to maintain stateful sessions with HTTP user agents, 
addressing some of the security and privacy considerations of HTTP Cookies. 
"[HTTP State Tokens](https://mikewest.github.io/http-state-tokens/draft-west-http-state-tokens.html)" 

*   The Tracking Protection Working Group discussed limits on all browser storage (identifiers) to a default short duration when DNT:1 was set
e.g. 
"[public-tracking@w3.org Mail Archives](https://lists.w3.org/Archives/Public/public-tracking/2013Jun/0262.html)" 

*   Safari's ITP 2.1 limits the duration of first-party cookies created by script to a default 7 days duration. 
"[Intelligent Tracking Prevention 2.1](https://webkit.org/blog/8613/intelligent-tracking-prevention-2-1/)"

*   Safari's version of the Storage Access API document.requestStorageAccess causes a user prompt. 
"[Introducing Storage Access API](https://webkit.org/blog/8124/introducing-storage-access-api/)"
  
