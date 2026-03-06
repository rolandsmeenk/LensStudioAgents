# ExamplePurchaseController Reference

```typescript
import { BaseButton } from "SpectaclesUIKit.lspkg/Scripts/Components/Button/BaseButton";
import { CommerceKit } from "CommerceKit.lspkg/CommerceKit";

@component
export class ExamplePurchaseController extends BaseScriptComponent {
  @input
  productId: string = "premium_upgrade"; // Set your product ID here

  @input
  purchaseButton: BaseButton;
  @input
  nameText: Text;
  @input
  priceText: Text;
  @input
  statusText: Text;

  private commerceKit: CommerceKit = CommerceKit.getInstance();

  private pendingPurchase: boolean = false;
  onAwake() {
    this.createEvent("OnStartEvent").bind(() => {
      this.onStart();
    });
  }

  private async onStart() {
    // Setup UI
    this.setupPurchaseButton();
    this.updateUI();
  }
  private setupPurchaseButton() {
    this.purchaseButton.onTriggerUp.add(() => {
      this.onPurchaseButtonPressed();
    });
  }
  private async updateUI() {
    await this.commerceKit.client;
    const isOwned = this.commerceKit.isProductOwned(this.productId);

    this.purchaseButton.enabled = !isOwned;
    this.statusText.text = isOwned ? "Premium Owned" : "Tap to Purchase";

    const product = await this.commerceKit.getProductInfo(this.productId);
    if (isNull(product)) {
      this.statusText.text = "Product not found";
    } else {
      this.nameText.text = product.displayName;
      this.priceText.text = `${product.price.price} ${product.price.currency}`;
    }
  }
  private async onPurchaseButtonPressed() {
    if (this.commerceKit.isProductOwned(this.productId)) {
      this.statusText.text = "Already owned!";
      return;
    }
    if (this.pendingPurchase) {
      return;
    }

    try {
      this.pendingPurchase = true;
      this.statusText.text = "Processing purchase...";

      const result = await this.commerceKit.purchaseProduct(this.productId);
      if (result.success) {
        this.statusText.text = "Purchase successful!";
        this.enablePremiumFeatures();
      } else if (result.cancelled) {
        this.statusText.text = "Purchase cancelled";
      } else if (!result.success) {
        this.statusText.text = "Purchase failed";
      }
      this.pendingPurchase = false;
    } catch (error) {
      this.statusText.text = `Purchase failed: ${error.message}`;
      print(`Purchase error: ${error}`);
      this.pendingPurchase = false;
    }
  }
  private enablePremiumFeatures() {
    // Enable your premium features here
    print("Premium features unlocked!");
  }
}

```

