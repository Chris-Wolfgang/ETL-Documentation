type: fix

Cookbook recipes now pass the `CancellationToken` to `TransformAsync`, and the transform-chain factory returns the cancellation-aware chain.
