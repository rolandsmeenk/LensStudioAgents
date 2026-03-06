# CommerceKit Reference

```typescript
import { CommercePrice } from "./Helper/CommercePrice";
import { CommerceProduct } from "./Helper/CommerceProduct";
import { Singleton } from "./Utils/Singleton";

interface PendingPurchase {
  resolve: (value: { success: boolean; cancelled?: boolean }) => void;
  reject: (reason: any) => void;
}

@Singleton
export class CommerceKit {
  /**
   * Singleton instance getter for CommerceKit.
   * Use this method to access the shared instance of CommerceKit.
   *
   * @returns The singleton instance of CommerceKit
   */
  public static getInstance: () => CommerceKit;
  private commerceModule =
    require("LensStudio:CommerceKitModule") as CommerceKitModule;

  private productCatalog: CommerceProduct[] = [];
  private commerceClient: Promise<Commerce.Client>;

  private pendingPurchases = new Map<string, PendingPurchase>();
  private ownedProducts = new Map<string, Commerce.PurchaseState>();
  /**
   * Initializes the product catalog and validates price formats.
   * This method must be called before making any purchases.
   *
   * @param productCatalog - The catalog of products to initialize
   * @throws Error if catalog price validation fails
   */
  public async initializeCatalog(productCatalog: CommerceProduct[]) {
    if (this.productCatalog.length > 0) {
      print("Product catalog has already been initialized.");
      return;
    }

    //validate the price format
    //then convert this into format that works for commerce kit. For example, $1.99 should be represented as 199.

    // First validate prices (ensure they're numbers with proper format)
    this.productCatalog = this.validatePriceFormat(productCatalog);

    // Then convert prices from decimal to cents (e.g., 2.99 to 299)
    const formattedCatalog = this.formatPriceFormat(this.productCatalog);

    print("Product catalog validated and formatted successfully.");

    const commerceClient = this.buildClient(formattedCatalog);

    this.commerceClient = new Promise<Commerce.Client>((resolve, reject) => {
      if (global.deviceInfoSystem.isEditor()) {
        this.editorRefreshPurchaseHistory();
        reject(
          "Commerce client is not available in editor. Mock flow initiated."
        );
      } else {
        commerceClient
          .startConnection()
          .then(async () => {
            print("Commerce client connection started successfully");
            try {
              // Query purchase history before resolving the promise
              const history = await commerceClient.queryPurchaseHistory();

              print("Refreshing purchase history: #" + history.length);
              this.ownedProducts.clear();
              history.forEach((purchase) => {
                print(
                  `Status: "${purchase.productId}": "${purchase.purchaseState}"`
                );
                this.ownedProducts.set(
                  purchase.productId,
                  purchase.purchaseState
                );
              });

              // Only resolve after history is successfully refreshed
              resolve(commerceClient);
            } catch (historyError) {
              print(`Failed to refresh purchase history: ${historyError}`);
              reject(historyError);
            }
          })
          .catch((error) => {
            print(`Failed to start commerce client connection: ${error}`);
            reject(error);
          });
      }
    });

    // Wait for the commerceClient Promise to resolve or reject
    try {
      await this.commerceClient;
      print("CommerceKit fully initialized successfully.");
    } catch (error) {
      print(`CommerceKit initialization failed: ${error}`);
    }
  }

  /**
   * Validates and formats the prices in the product catalog.
   * Ensures prices are properly formatted with two decimal places and not negative.
   *
   * @param productCatalog - The catalog of products to validate
   * @returns The validated product catalog
   * @throws Error if any product has a negative price
   */
  private validatePriceFormat(
    productCatalog: CommerceProduct[]
  ): CommerceProduct[] {
    if (!productCatalog || productCatalog.length === 0) {
      print("Warning: Empty product catalog provided.");
      return [];
    }

    // Create a deep copy of the product catalog to avoid modifying the original objects
    const validatedCatalog = productCatalog.map((product) => {
      // Create a deep copy of the product
      const productCopy = { ...product, price: { ...product.price } };

      if (!productCopy.price) {
        throw new Error(
          `Product ${productCopy.id} is missing price information.`
        );
      }

      // Ensure price is a number
      if (typeof productCopy.price.price !== "number") {
        throw new Error(
          `Product ${productCopy.id} has an invalid price format.`
        );
      }

      // Check for negative price
      if (productCopy.price.price < 0) {
        throw new Error(
          `Product ${productCopy.id} has a negative price: ${productCopy.price.price}. Negative prices are not allowed.`
        );
      }

      // Format the price to have exactly two decimal places (e.g., 2.999999 becomes 2.99)
      // Use Math.round to ensure proper rounding
      productCopy.price.price = Math.round(productCopy.price.price * 100) / 100;

      return productCopy;
    });

    return validatedCatalog;
  }

  /**
   * Converts decimal prices to integer cents format.
   * For example: $2.99 becomes 299, $10.00 becomes 1000.
   *
   * @param productCatalog - The catalog of products to format
   * @returns The formatted product catalog with prices converted to cents
   */
  private formatPriceFormat(
    productCatalog: CommerceProduct[]
  ): CommerceProduct[] {
    if (!productCatalog || productCatalog.length === 0) {
      return [];
    }

    // Create a deep copy of the product catalog
    const formattedCatalog = productCatalog.map((product) => {
      // Create a deep copy of the product
      const productCopy = { ...product, price: { ...product.price } };

      switch (productCopy.price.currency) {
        case "USD":
          (productCopy.price as any).displayPrice =
            "$" + productCopy.price.price.toString();
          break;
        default:
          throw new Error(
            "Unsupported currency: " + productCopy.price.currency
          );
      }

      // Convert the price from dollars to cents (multiply by 100)
      // Since validatePriceFormat already ensures the price is properly formatted with two decimal places,
      // we can simply multiply by 100 and round to get an integer value in cents
      productCopy.price.price = Math.round(productCopy.price.price * 100);

      return productCopy;
    });

    return formattedCatalog;
  }

  private buildClient(productCatalog: CommerceProduct[]): Commerce.Client {
    const clientOptions = new Commerce.ClientOptions((result, purchases) => {
      this.onPurchasesUpdated(result, purchases);
    });

    // this.fillInDisplayPrice(productCatalog);
    clientOptions.extras = JSON.stringify({
      productCatalog: productCatalog,
    });

    return this.commerceModule.createClient(clientOptions);
  }

  private editorRefreshPurchaseHistory(): void {
    this.ownedProducts.clear();
    this.productCatalog.forEach((product) => {
      this.ownedProducts.set(product.id, Commerce.PurchaseState.Unset);
    });
  }
  private async refreshPurchaseHistory(): Promise<void> {
    const commerceClient = await this.commerceClient;
    const history = await commerceClient.queryPurchaseHistory();

    print("Refreshing purchase history: #" + history.length);
    this.ownedProducts.clear();
    history.forEach((purchase) => {
      print(`Status: "${purchase.productId}": "${purchase.purchaseState}"`);
      this.ownedProducts.set(purchase.productId, purchase.purchaseState);
    });
  }

  private onPurchasesUpdated(
    result: Commerce.PurchaseResult,
    purchases: Commerce.Purchase[]
  ): void {
    if (!purchases?.length) {
      print("No purchases to process.");
      return;
    }

    switch (result.responseCode) {
      case Commerce.ResponseCode.Success:
        this.onSuccessfulPurchase(purchases);
        break;
      case Commerce.ResponseCode.UserCanceled:
        this.onUserCancelledPurchase(purchases);
        break;
      default:
        this.onPurchaseError(result, purchases);
        break;
    }
  }

  private async onSuccessfulPurchase(
    purchases: Commerce.Purchase[]
  ): Promise<void> {
    const commerceClient = await this.commerceClient;

    purchases.forEach(async (purchase) => {
      const context = this.pendingPurchases.get(purchase.productId);
      if (!context || !purchase.token) return;

      try {
        const ackResult = await commerceClient.acknowledgePurchase(
          purchase.token
        );
        await this.refreshPurchaseHistory();

        if (ackResult.responseCode === Commerce.ResponseCode.Success) {
          context.resolve({ success: true });
        } else {
          context.reject(
            new Error(`Acknowledgment failed: ${ackResult.responseCode}`)
          );
        }

        this.pendingPurchases.delete(purchase.productId);
      } catch (error) {
        print(`Failed to acknowledge purchase: ${error.message || error}`);
      }
    });
    print("Purchase updates received successfully.");
  }

  private async onUserCancelledPurchase(
    purchases: Commerce.Purchase[]
  ): Promise<void> {
    print("Purchase was cancelled.");

    // Refresh states even after cancellation to ensure consistency
    await this.refreshPurchaseHistory();

    purchases.forEach((purchase) => {
      const context = this.pendingPurchases.get(purchase.productId);
      if (context) {
        context.resolve({ success: false, cancelled: true });
        this.pendingPurchases.delete(purchase.productId);
      } else {
        print(
          `No purchase context found for cancelled product: ${purchase.productId}`
        );
      }
    });
  }

  private async onPurchaseError(
    result: Commerce.PurchaseResult,
    purchases: Commerce.Purchase[]
  ): Promise<void> {
    print(
      `Purchase error: ${result.responseCode}, purchases: {${purchases
        .map((p) => p.productId)
        .join(", ")}}`
    );

    // Refresh states after error to ensure consistency
    await this.refreshPurchaseHistory();

    purchases.forEach((purchase) => {
      print(`Failed to purchase: ${purchase.productId}`);
      const context = this.pendingPurchases.get(purchase.productId);
      if (context) {
        context.reject(
          new Error(
            "Purchase error: " +
              result.responseCode +
              ", " +
              (result.debugMessage || "")
          )
        );
        this.pendingPurchases.delete(purchase.productId);
      }
    });
  }

  /**
   * Returns the commerce client instance.
   *
   * @returns A promise that resolves to the Commerce.Client instance
   * @throws Error if the client failed to initialize
   */
  public async client(): Promise<Commerce.Client> {
    return await this.commerceClient;
  }

  /**
   * Initiates the purchase flow for a product.
   *
   * @param productId - The unique identifier of the product to purchase
   * @returns A promise that resolves to an object indicating purchase success or cancellation
   * @throws Error if the purchase acknowledgment fails or if the purchase flow fails to launch
   */
  public async purchaseProduct(
    productId: string
  ): Promise<{ success: boolean; cancelled?: boolean }> {
    // Check if already owned
    const ownedState = this.ownedProducts.get(productId);
    if (ownedState === Commerce.PurchaseState.Purchased) {
      return { success: true };
    }

    if (global.deviceInfoSystem.isEditor()) {
      if (!isNull(this.ownedProducts.get(productId))) {
        print(`Mock purchase of product ${productId} in editor.`);
        this.ownedProducts.set(productId, Commerce.PurchaseState.Purchased);
        return { success: true };
      } else {
        print(`Product ${productId} doesn't exist in catalog.`);
        return { success: false };
      }
    }

    const commerceClient = await this.commerceClient;
    if (ownedState === Commerce.PurchaseState.Pending) {
      print(`Product ${productId} is pending, attempting to acknowledge.`);
      const result = await commerceClient.acknowledgePurchase(productId);

      if (result.responseCode === Commerce.ResponseCode.Success) {
        print(`Product ${productId} acknowledged successfully.`);
        return { success: true };
      }
      throw new Error(
        `Failed to acknowledge pending purchase: ${result.responseCode}`
      );
    }
    try {
      const result = await commerceClient.launchPurchaseFlow(productId);
      if (result.responseCode === Commerce.ResponseCode.Success) {
        return new Promise((resolve, reject) => {
          this.pendingPurchases.set(productId, { resolve, reject });
        });
      } else {
        throw new Error(`Purchase failed: ${result.responseCode}`);
      }
    } catch (e) {
      print(e);
      return;
    }
  }

  /**
   * Checks if a product is already owned by the user.
   *
   * @param productId - The unique identifier of the product to check
   * @returns true if the product is owned, false otherwise
   */
  public isProductOwned(productId: string): boolean {
    return (
      this.ownedProducts.get(productId) === Commerce.PurchaseState.Purchased
    );
  }

  /**
   * Retrieves detailed information about a product.
   * In editor mode, it returns the product from the local catalog.
   * In runtime mode, it queries the commerce client for product details.
   *
   * @param productId - The unique identifier of the product
   * @returns A promise that resolves to a CommerceProduct containing product details
   * @throws Error if the product is not found
   */
  public async getProductInfo(productId: string): Promise<CommerceProduct> {
    if (global.deviceInfoSystem.isEditor()) {
      const purchase = this.productCatalog.find((p) => p.id === productId);
      return purchase;
    }
    const commerceClient = await this.commerceClient;
    const details = await commerceClient.queryProductDetails([productId]);

    if (!details || details.length === 0) {
      throw new Error(`Product ${productId} not found`);
    }

    const product = details[0];
    const convertedProduct = new CommerceProduct();
    convertedProduct.id = product.id;
    switch (product.type) {
      case Commerce.ProductType.NonConsumable:
        convertedProduct.type = "NonConsumable";
        break;
      default:
        convertedProduct.type = "Unknown";
    }
    convertedProduct.displayName = product.displayName;
    const price = new CommercePrice();
    price.currency = product.price.currency;
    price.price = product.price.price / 100; // Convert cents back to dollars
    convertedProduct.price = price;
    return convertedProduct;
  }

  /**
   * Checks the current purchase status of a product.
   * In editor mode, it returns the status from the local purchase history.
   * In runtime mode, it queries the commerce client for purchase history.
   *
   * @param productId - The unique identifier of the product to check
   * @returns A promise that resolves to the Commerce.PurchaseState of the product
   */
  public async checkPurchaseStatus(
    productId: string
  ): Promise<Commerce.PurchaseState> {
    if (global.deviceInfoSystem.isEditor()) {
      return this.ownedProducts.get(productId)
        ? this.ownedProducts.get(productId)
        : Commerce.PurchaseState.Unset;
    }
    const commerceClient = await this.commerceClient;
    const history = await commerceClient.queryPurchaseHistory();

    const purchase = history.find((p) => p.productId === productId);
    return purchase ? purchase.purchaseState : Commerce.PurchaseState.Unset;
  }
}

```

