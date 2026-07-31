---
name: Issue and transfer a game item (Saga)
description: >-
  Mint an item from an item type and move it into a player's wallet, then transfer it
  to another player, using the Mythical Saga gRPC API. Handles the ack-then-stream
  async write pattern.
api: grpc/saga/
operations: [CreateItemType, StartMint, IssueItem, GetItemsForPlayer, TransferItem]
---

# Issue and transfer a game item

Use the Saga SDK (Java/C++/Python) — this is a gRPC/Protobuf API, not REST. Credentials
are supplied via the SDK environment config and requests are scoped by `title_id` /
`game_title_id`. See `authentication/mythical-games-authentication.yml` and
`conventions/mythical-games-conventions.yml`.

## Steps

1. **Define the item template.** Call `ItemTypeService.CreateItemType`
   (`grpc/saga/api/itemtype/rpc.proto`) to register the item type, then
   `StartMint` to open a minting window (close later with `EndMint`).
2. **Issue an item to a player.** Call `ItemService.IssueItem`
   (`grpc/saga/api/item/rpc.proto`). Like all writes it returns
   `ReceivedResponse{trace_id}` immediately — the mint is not final yet.
3. **Await finalization.** Watch `StatusStream` (`grpc/saga/streams/stream.proto`)
   for the terminal `StatusUpdate` correlated by `trace_id`, then acknowledge with
   `StatusConfirmation`. Do not treat the item as owned until this arrives.
4. **Confirm ownership.** Call `ItemService.GetItemsForPlayer` to verify the item is
   in the player's inventory.
5. **Transfer.** Call `ItemService.TransferItem` (or `TransferItemBulk`) — again
   ack-then-stream; reconcile via the status stream.

## Rules
- No idempotency key exists; guard against duplicate writes yourself and reconcile by
  `trace_id`. See `conventions/mythical-games-conventions.yml`.
- Errors arrive as gRPC status plus `ErrorProto`/`ErrorData`
  (`errors/mythical-games-error-codes.yml`).
