---
layout: post
categories: notes devops
title: "How to Hide Modal App Behind Cloudflare Zero Trust"
description: "How to Hide Modal Behind Cloudflare Zero Trust"
keywords: "devops, modal"
---

## Why you might need this

- To remove your app from public access without implementing auth in code
- To integrate the app with your company's existing access model
- To use Google (or another provider) auth on the cheap

## Downsides

- This is not suitable for production or high load, since cloudflared isn't designed for heavy traffic and will become a bottleneck
- Latency will increase, as traffic will follow the additional hops: Cloudflare Edge Servers → Your cloudflared installation → Modal

## Prerequisites

- You have a Team or Enterprise plan on Modal (required for custom domain support)
- It's assumed you have cloudflared deployed and everything set up in Cloudflare Zero Trust and you're familiar with Cloudflare Zero Trust concepts

## Algorithm

1. Attach your custom domain to Modal and verify it (<https://modal.com/docs/guide/webhook-urls#custom-domains>). This is needed so Modal's infrastructure "sees" your domain and you can use it as an entry point.
2. Deploy the app with the custom domain and make sure everything works (without auth for now).
3. Create a Cloudflare Zero Trust Application: specify access policies, auth method, and the app domain (same as your custom domain), then bind it to the tunnel.
4. In the tunnel settings, add a route from `your_custom_domain` to Modal's standard web endpoint (`https://<source>--<label>.modal.run`).
5. If your app's cold start is fairly long, consider bumping the Connect Timeout in the route settings from the default 30 seconds.
6. In your DNS provider, change the CNAME record from `your_custom_domain: cname.modal.domains` to `your_custom_domain: cf_tunnel_id.cfargotunnel.com`.

After this, your app will be accessible via the standard URL `https://<source>--<label>.modal.run` without auth, and via `https://your_custom_domain` with configured auth policies. To disable access through the standard URL, you can add a redirect in the app code. Check the `host` HTTP header, and if it doesn't match `your_custom_domain`, then redirect to it. You can also handle `modal serve` development mode there.

```python
CUSTOM_DOMAIN = os.environ.get("MODAL_CUSTOM_DOMAIN", "modal-demo.your-domain.com")
<...>
    @modal.asgi_app(custom_domains=[CUSTOM_DOMAIN])
    def serve(self):
        web = FastAPI(title="Demo (Modal)")

        # Redirect to custom domain if configured.
        # Skip redirect for ephemeral apps (modal serve) — their hosts end with "-dev.modal.run".
        _redirect_enabled = bool(CUSTOM_DOMAIN)

        if _redirect_enabled or HEADERS_DEBUG:
            _allowed_hosts = {CUSTOM_DOMAIN} if _redirect_enabled else set()

            @web.middleware("http")
            async def _domain_middleware(request: Request, call_next):
                if _redirect_enabled:
                    host = request.headers.get("host", "").split(":")[0]
                    is_ephemeral = host.endswith("-dev.modal.run")
                    if not is_ephemeral and host not in _allowed_hosts:
                        target = request.url.replace(
                            scheme="https", hostname=CUSTOM_DOMAIN, port=None,
                        )
                        return RedirectResponse(str(target), status_code=301)

                return await call_next(request)
```

## How it works

*I don't have exact knowledge of Modal's internal infrastructure, so this is a hypothesis that may contain inaccuracies.*

When we create a Modal app with custom_domains attached, Modal's infrastructure gets instructed that incoming traffic with the `host: your_custom_domain` header should be routed to our app. With the default DNS settings `your_custom_domain: cname.modal.domains`, this follows the route `browser → cname.modal.domains → app`. If we change the DNS record, the route gets longer: `browser → Cloudflare network → our cloudflared installation → app`, but the host header stays the same, so Modal's infrastructure still routes traffic to the correct app.
