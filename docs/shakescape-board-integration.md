# ShakeScape board on Liquidity Spot

## Goal and current boundary

Show live, signed HNS/BTC ShakeScape offers on Liquidity Spot without presenting
them as Liquidity Spot orders. The first integration is read-only. A visitor can
inspect terms and the offer ID, then use ShakeScape to act on the offer. The site
must not create a local `Order` or `Swap`, reserve funds, or claim that its
`/orders/<id>/accept` flow can accept a ShakeScape offer.

Today `/orders` and `/api/channel` read open rows from Liquidity Spot's `Order`
table. Those rows have a site user ID and are accepted through a site-owned swap
room. ShakeScape instead gossips `DirectOffer` objects between authenticated
peers. A `DirectOffer` has a signed offer ID, network binding, exact offered and
received assets and base-unit amounts, and an expiry. It is indivisible: a
displayed price level must not imply a partial fill. Its offer setter becomes
the protocol taker when another wallet accepts it; this differs from the
site's current Alice/Bob order flow.

The hsrd relay already verifies the offer ID, signature, network, and active
window, and applies signed cancellation tombstones. Its loopback wallet RPC
has a paged name-market snapshot, but that route requires the wallet index
profile and is unavailable on a relay-only node. There is no cross-chain offer
snapshot. Liquidity Spot does not run hsrd: its chain watchers make optional
HTTP transaction lookups and do not participate in Handshake P2P gossip.

## Smallest useful data path

1. Add a bounded read-only cross-chain snapshot to hsrd's existing relay
   handle, parallel to its name-market snapshot. Serve it through one new
   read-only route on hsrd's existing loopback HTTP RPC listener, independently
   of the wallet index profile. Return only offers that passed normal
   `DirectOffer::verify_at` admission and remain unexpired and uncancelled.
   Include a snapshot revision or generation marker so a reader never combines
   different inventories. Prefer a private connection from Liquidity Spot to
   this route. If the deployments cannot share a private network, put a narrow
   authenticated HTTPS gateway in front of this one route on the existing hsrd
   host. Never expose the broad wallet RPC or run a second full hsrd node
   inside the web app.
2. Add a server-side reader in `services/shakescape_board.py`. Its source URL is
   deployment configuration, never a request parameter. Fetch with a short
   timeout, a response-size and offer-count limit, no redirects, and a small
   cache. Reject malformed JSON, an unexpected mainnet binding, duplicate IDs,
   impossible amounts, and expired offers. If the feed fails or becomes stale,
   show the board as temporarily unavailable rather than reusing old offers as
   active listings. No browser request goes directly to the relay.
3. Normalize each row to a display-only `ExternalOffer` with source
   `shakescape`, lowercase hex `offer_id`, exact HNS/BTC base-unit amounts,
   signer identity fingerprint, creation and expiry times, and feed timestamp.
   Convert HNS and BTC with their respective chain units using decimal
   arithmetic. If HNS is offered, label the side **sell HNS**; if HNS is
   received, label it **buy HNS**. Show the exact BTC total and derived BTC/HNS
   price, and state that the amount is one whole offer.
4. Add a separate public `/shakescape` board page and a versioned
   `/api/external-offers/shakescape` read endpoint. Link to the page from
   `/orders`, but leave the local order table, `POST /orders`,
   `/orders/<int:order_id>/accept`, and the current `/api/channel` v1 payload
   unchanged. Use an offer ID route only for read-only detail; it must not
   resolve a local integer `Order` ID. The initial action is **Open
   ShakeScape**, with a copyable offer ID and instructions to check the current
   terms in the app. Add a direct offer link only after the mobile app supports a
   canonical offer URI and re-verifies that exact offer on open.

An accepted offer may still be advertised briefly by a relay, so the site
should call a row **advertised on the board**, not **reserved** or **available
to settle**. ShakeScape remains the authority for current offer terms,
acceptance, wallet approval, and settlement. No private swap-session messages,
addresses, keys, or transaction history enter the public feed.

## Verification before enabling the page

- A valid HNS-for-BTC and BTC-for-HNS offer display with the correct side,
  exact amounts, decimal price, network, expiry, and source identity.
- Cancellation, expiry, and feed loss remove or disable the row promptly;
  malformed, cross-network, and duplicate offers never render as active.
- An external offer ID cannot reach the local order acceptance or site swap
  creation routes. Existing `/orders` and `/api/channel` v1 tests still pass.
- Bounded feed reads do not block ordinary P2P browsing when the bridge is
  offline. Review the actual page labels and links on desktop and mobile.

Build the hsrd snapshot and its feed contract first, then the Liquidity Spot
reader and page against a fixed snapshot fixture. Turn on the page only after
the deployed feed passes a live readback with at least one known signed offer
and a cancellation/expiry check.
