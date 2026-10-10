# Partner legacy API retirement plan

Not executed. The legacy routes stay. The web portal no longer calls them. That is not enough evidence to delete a public API.

How long to watch production traffic is an operational choice. This document does not set that period.

Removal later requires all of the following:

- the web client has no remaining calls
- no documented external consumer
- production usage telemetry (`partner.legacy_endpoint.used`) shows no calls for the observation period operations chooses

## Inventory

| Legacy route | Web calls it | Replacement | OpenAPI | Tests |
| --- | --- | --- | --- | --- |
| `GET /api/v1/orgs/partner/summary/` | No | `GET /api/v1/partner/channels/{channel_id}/summary/` | Yes | `test_partner_summary_api.py` |
| `GET /api/v1/orgs/partner/customers/` | No | `GET /api/v1/partner/channels/{channel_id}/customers/` | Yes | `test_partner_customers_api.py` |
| `GET /api/v1/orgs/partner/grants/` | No | `GET /api/v1/partner/channels/{channel_id}/grants/` | Yes | `test_partner_grants_api.py` |
| `GET /api/v1/orgs/partner/invite-link/` | No | `GET /api/v1/partner/channels/{channel_id}/invite-link/` | Yes | `test_partner_invite_flow.py` |
| `POST .../invite-link/regenerate/` | No | `POST /api/v1/partner/channels/{channel_id}/invite-link/regenerate/` | Yes | invite flow tests |
| `POST .../invite-link/activate/` | No | `POST /api/v1/partner/channels/{channel_id}/invite-link/activate/` | Yes | invite flow tests |
| `POST .../invite-link/deactivate/` | No | `POST /api/v1/partner/channels/{channel_id}/invite-link/deactivate/` | Yes | invite flow tests |
| `POST /api/v1/billing/partner-grants/` | No | `POST /api/v1/partner/channels/{channel_id}/grants/` | Yes | `test_partner_grant_api.py` |

Internal usage that remains on purpose: the API tests above, and any operator or script still using the legacy URLs. Each legacy view logs `partner.legacy_endpoint.used` with the endpoint name, method, user id, and status class. It does not log the body, a customer id, or an invite token.

The legacy routes still resolve a single team channel. They do not gain individual or multi-context behavior. Several team channels still return `partner_context_ambiguous` on those old URLs. The new portal does not use that response.
