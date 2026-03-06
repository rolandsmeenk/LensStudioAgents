---
name: spectacles-commerce
description: Reference guide for CommerceKit and In-App Purchases on Spectacles — covering CommerceKitModule initialization, converting prices to integer cents, editor mock purchases versus device production purchases, queryPurchaseHistory for ownership checking, launchPurchaseFlow to prompt a swipe-to-buy, acknowledgePurchase to verify consumption, and handling UserCanceled responses. Use this skill when building premium lenses, freemium features, unlocking levels, or selling non-consumable digital items on the Snap platform.
---

# Spectacles CommerceKit — Reference Guide

**CommerceKit** enables handling in-app purchases (IAP) for **non-consumable** digital items directly inside Custom Lenses on Spectacles.

> **Note:** Commerce capabilities are in Closed Beta and require your Lens Studio account to have proper onboarding.

---

## Setup & Architecture

Commerce uses the `CommerceKitModule`. You define a local "Product Catalog" representing the items you are selling. 

*Prices in CommerceKit APIs are always expected in Integer Cents (e.g. `$2.99` is passed as `299`).*

---

## 1. Creating the Commerce Client

Initialize the client with your catalog. You must provide a callback to handle asynchronous purchase results.

```typescript
const commerceModule = require("LensStudio:CommerceKitModule") as CommerceKitModule
let commerceClient: Commerce.Client

// Define your products
const catalog = [
  { id: "premium_skin_01", displayName: "Gold Skin", price: { currency: "USD", price: 299 } },
  { id: "level_pack_02", displayName: "Extra Levels", price: { currency: "USD", price: 499 } }
]

const clientOptions = new Commerce.ClientOptions((result, purchases) => {
  onPurchasesUpdated(result, purchases)
})

// Embed the catalog as JSON in the extras
clientOptions.extras = JSON.stringify({ productCatalog: catalog })

// Create client
commerceClient = commerceModule.createClient(clientOptions)

// Connect to the commerce backend
commerceClient.startConnection()
  .then(async () => {
    print("Commerce connection started.")
    await refreshOwnership()
  })
  .catch((e) => print("Commerce connection failed: " + e))
```

---

## 2. Checking Ownership History

Before offering a purchase, query the user's history to see if they already own it.

```typescript
const ownedProducts = new Map<string, Commerce.PurchaseState>()

async function refreshOwnership() {
  const history = await commerceClient.queryPurchaseHistory()
  
  ownedProducts.clear()
  history.forEach((purchase) => {
    // purchaseState can be: Purchased, Pending, Unset
    ownedProducts.set(purchase.productId, purchase.purchaseState)
    print(`Product ${purchase.productId} is ${purchase.purchaseState}`)
  })
}

function isProductOwned(productId: string): boolean {
  return ownedProducts.get(productId) === Commerce.PurchaseState.Purchased
}
```

---

## 3. Launching the Purchase Flow

When a user taps a "Buy" button, attempt to launch the flow. 

```typescript
async function buyProduct(productId: string) {
  // 1. Editor mock check
  if (global.deviceInfoSystem.isEditor()) {
    print(`Mock purchase completed for ${productId} (Editor mode)`)
    ownedProducts.set(productId, Commerce.PurchaseState.Purchased)
    return
  }

  // 2. Already owned?
  if (isProductOwned(productId)) {
    print("User already owns this.")
    return
  }
  
  // 3. Launch flow
  const result = await commerceClient.launchPurchaseFlow(productId)
  if (result.responseCode !== Commerce.ResponseCode.Success) {
    print("Failed to launch flow: " + result.responseCode)
  }
  // The result will bubble up to the onPurchasesUpdated callback.
}
```

---

## 4. Handling the Purchase Callback

The `ClientOptions` callback receives results from the system purchase overlay. A purchase isn't finalized until you **acknowledge** it.

```typescript
async function onPurchasesUpdated(result: Commerce.PurchaseResult, purchases: Commerce.Purchase[]) {
  if (!purchases || purchases.length === 0) return

  switch (result.responseCode) {
    case Commerce.ResponseCode.Success:
      for (const p of purchases) {
         if (p.token) await finalizePurchase(p.token)
      }
      break
      
    case Commerce.ResponseCode.UserCanceled:
      print("User cancelled the purchase flow.")
      break
      
    default:
      print(`Purchase error: Code ${result.responseCode}`)
      break
  }
  
  // Always refresh logic to ensure sync
  await refreshOwnership()
}
```

### Acknowledging the Purchase

```typescript
async function finalizePurchase(token: string) {
  try {
    const ackResult = await commerceClient.acknowledgePurchase(token)
    if (ackResult.responseCode === Commerce.ResponseCode.Success) {
      print("Purchase verified and acknowledged successfully!")
      // Unlock content in the lens here
    }
  } catch (err) {
    print("Failed to acknowledge purchase: " + err)
  }
}
```

---

## Common Gotchas

- **Cent conversion**: Always multiply floating-point dollars by 100 and use `Math.round()` prior to passing to CommerceKit to avoid floating point drift. (e.g. `Math.round(2.99 * 100)`).
- **Editor Mode**: Commerce flows do *not* launch in Lens Studio editor. Mock your purchases by checking `global.deviceInfoSystem.isEditor()` and faking an immediate "Owned" state.
- **Pending States**: `queryPurchaseHistory` might return `Commerce.PurchaseState.Pending` if network failed halfway. You should attempt to re-acknowledge pending items on startup.

---

## Reference Examples
*   [CommerceKit.ts](references/CommerceKit.md) - Core singleton client wrapping `CommerceKitModule`.
*   [ExamplePurchaseController.ts](references/ExamplePurchaseController.md) - A comprehensive implementation example.

