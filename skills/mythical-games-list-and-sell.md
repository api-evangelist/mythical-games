---
name: List an item and complete a sale (Saga marketplace)
description: >-
  Create a marketplace listing for an owned item, accept an offer, and settle the
  trade using the Mythical Saga gRPC API.
api: grpc/saga/
operations: [CreateListingQuote, ConfirmListing, GetListings, CreateOfferQuote, ConfirmOffer]
---

# List an item and complete a sale

Marketplace flows in Saga are quote-then-confirm and use the same ack-then-stream
async pattern as all writes. Access is via the SDK; requests are title-scoped.

## Steps

1. **Quote a listing.** Call `ListingService.CreateListingQuote`
   (`grpc/saga/api/listing/rpc.proto`) for the item and price. Returns
   `ReceivedResponse{trace_id}`.
2. **Confirm the listing.** Call `ListingService.ConfirmListing` to publish it.
   Reconcile finalization via `StatusStream` (`grpc/saga/streams/stream.proto`).
3. **Discover listings.** Buyers browse with `ListingService.GetListings`
   (cursor pagination: `page_size` + `created_at_cursor`, see
   `conventions/mythical-games-conventions.yml`).
4. **Make an offer.** Call `OfferService.CreateOfferQuote`
   (`grpc/saga/api/offer/rpc.proto`), then `OfferService.ConfirmOffer` to accept.
5. **Cancel path.** Use `ListingService.CancelListing` or
   `OfferService.CancelOffer` to back out before settlement.

## Rules
- Currency movement uses `CurrencyService` (`GetBalanceOfPlayer`, `TransferCurrency`).
- Every mutating call is asynchronous — treat a returned `trace_id` as "received",
  not "settled". Correlate the final outcome on the status stream.
- No idempotency key; dedupe on your side.
