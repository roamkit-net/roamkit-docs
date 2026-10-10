# Partner channels: individual sellers and multiple contexts

A partner can now be a person or a team.

An individual partner uses the personal account they already have. Commission from their customers and credit they grant to those customers use that same balance. A team partner still uses the organization account. Neither kind converts an account from personal to organization, or the other way around.

The partner portal lists every channel the signed-in user may open. One channel opens directly. Several channels ask which one to use, and the portal remembers that choice in the browser. Summary, customers, grant history, grants, and the invite link all name the channel in the request. A viewer can read. An owner or team admin can grant credit. Only an owner can change the invite link.

A person cannot be a customer of a channel they own or of a team they actively belong to. A purchase in that situation does not pay partner commission.

The previous partner URLs remain available for older clients. New portal builds do not call them.

Shipping order: the API, including database migration `0012_partner_channel_kind`, goes out before the web portal that depends on the new URLs. See [partner channel rollout](./partner-channel-rollout.md).
