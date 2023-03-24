# Implementing a store in your app using the StoreKit API

Offer in-app purchases and manage entitlements using signed transactions and status information.

## Overview

- Note:
  This sample code project is associated with WWDC22 session [110404: Implement proactive in-app purchase restore](https://developer.apple.com/wwdc22/110404/).
  
  It's also associated with WWDC21 session [10114: Meet StoreKit 2](https://developer.apple.com/wwdc21/10114/).

## Configure the sample code project

This sample code project uses StoreKit testing in Xcode so you can build and run the sample app without completing any setup in App Store Connect. The project defines in-app products for the StoreKit testing server in the `Products.storekit` file. The project includes the `Products.plist` as a resource file, which contains product identifiers that map to emoji characters.

By default, StoreKit testing in Xcode is in a disable state. Follow these steps to select the `Products.storekit` configuration file and enable StoreKit testing in Xcode:
1. Click the scheme to open the Scheme menu and choose Edit Scheme.
2. In the scheme editor, select the Run action.
3. Click Options in the action settings.
4. For the StoreKit Configuration option, select the `Products.storekit` configuration file.

When the app initializes a store, the system reads `Products.plist` and uses the product identifiers to request products from the StoreKit testing server.
# WWDC-STOREKIT2


# Links

https://developer.apple.com/documentation/storekit/in-app_purchase/implementing_a_store_in_your_app_using_the_storekit_api

https://developer.apple.com/videos/play/wwdc2021/10114/

https://www.revenuecat.com/blog/engineering/ios-in-app-subscription-tutorial-with-storekit-2-and-swift/

https://www.notion.so/StoreKit2-cd3b3114f048415f91d32193e518b1fa?pvs=4

# My Version

```swift
import Foundation
import StoreKit

public enum StoreError: Error {
    case failedVerification
}


@MainActor
final class PurchaseManager: ObservableObject {
    
    var isSubscribed: Bool {
        self.subscriptionGroupState == .subscribed || self.purchasedNonConsumable.count > 0
    }
    
    @Published private(set) var subscriptions: [Product] = []
    @Published private(set) var nonConsumables: [Product] = []
    @Published private(set) var purchasedSubscriptions: [Product] = []
    @Published private(set) var purchasedNonConsumable: [Product] = []
    @Published private(set) var subscriptionGroupState: Product.SubscriptionInfo.RenewalState?
    @Published private(set) var discountable: Bool = false
    
    var monthlySubscription: Product {
        self.subscriptions.first(where: { $0.id == Config.ProductIdentifier.oneMonthSubscription.rawValue })!
    }
    
    var yearlySubscription: Product {
        self.subscriptions.first(where: { $0.id == Config.ProductIdentifier.oneYearSubscription.rawValue })!
    }
    
    var specialYearlySubscription: Product {
        self.subscriptions.first(where: { $0.id == Config.ProductIdentifier.oneYearDiscountedSubscription.rawValue })!
    }
    
    var freeForever: Product {
        self.nonConsumables.first(where: { $0.id == Config.ProductIdentifier.freeForever.rawValue })!
    }
    
    private var updateListenerTask: Task<Void, Error>? = nil
    
    init() {
        self.updateListenerTask = self.listenForTransactions()
        Task {
            await self.fetchProducts()
            do {
                try await self.updateCustomerProductStatus()
            } catch {
                Log.error(error)
            }
        }
    }
    
    private func listenForTransactions() -> Task<Void, Error> {
        return Task.detached {
            //Iterate through any transactions that don't come from a direct call to `purchase()`.
            for await result in Transaction.updates {
                do {
                    let transaction = try await self.checkVerified(result)

                    //Deliver products to the user.
                    try await self.updateCustomerProductStatus()

                    //Always finish a transaction.
                    await transaction.finish()
                } catch {
                    //StoreKit has a transaction that fails verification. Don't deliver content to the user.
                    Log.error(error)
                }
            }
        }
    }
    
    deinit {
        self.updateListenerTask?.cancel()
    }
    
    func fetchProducts() async {
        do {
            let storeProducts = try await Product.products(for: Config.ProductIdentifier.all)
            Log.info("Fetched Products")
            Log.info(storeProducts.map(\.id))
            for product in storeProducts {
                switch product.type {
                case .nonConsumable:
                    self.nonConsumables.append(product)
                case .autoRenewable:
                    self.subscriptions.append(product)
                default: continue
                }
            }
            await self.checkEligibility()
        } catch {
            Log.error(error)
        }
    }
    
    @discardableResult
    func purchase(_ product: Product) async throws -> Transaction? {
        //Begin purchasing the `Product` the user selects.
        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            //Check whether the transaction is verified. If it isn't,
            //this function rethrows the verification error.
            let transaction = try checkVerified(verification)

            //The transaction is verified. Deliver content to the user.
            try await self.updateCustomerProductStatus()

            //Always finish a transaction.
            await transaction.finish()

            return transaction
        case .userCancelled, .pending:
            return nil
        default:
            return nil
        }
    }
    
    func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        //Check whether the JWS passes StoreKit verification.
        switch result {
        case .unverified:
            //StoreKit parses the JWS, but it fails verification.
            throw StoreError.failedVerification
        case .verified(let safe):
            //The result is verified. Return the unwrapped value.
            return safe
        }
    }

    func updateCustomerProductStatus() async throws {
        var purchasedNonConsumable: [Product] = []
        var purchasedSubscriptions: [Product] = []
        //Iterate through all of the user's purchased products.
        for await result in Transaction.currentEntitlements {
            //Check whether the transaction is verified. If it isn’t, catch `failedVerification` error.
            let transaction = try self.checkVerified(result)
            
            //Check the `productType` of the transaction and get the corresponding product from the store.
            switch transaction.productType {
            case .nonConsumable:
                if let nonConsumable = self.nonConsumables.first(where: { $0.id == transaction.productID }) {
                    purchasedNonConsumable.append(nonConsumable)
                }
            case .autoRenewable:
                if let subscription = self.subscriptions.first(where: { $0.id == transaction.productID }) {
                    purchasedSubscriptions.append(subscription)
                }
            default:
                break
            }
        }
        
        //Update the store information with the purchased products
        self.purchasedNonConsumable = purchasedNonConsumable
        self.purchasedSubscriptions = purchasedSubscriptions
        self.subscriptionGroupState = try await self.subscriptions.first?.subscription?.status.first?.state
    }
    
    func restore() async throws {
        try await AppStore.sync()
        try await self.updateCustomerProductStatus()
    }
    
    func isEligibleForIntroductoryPrice(product: Product) async -> Bool  {
        return await product.subscription?.isEligibleForIntroOffer ?? false
    }
    
    func checkEligibility() async {
        let monthlyEligibility = await self.isEligibleForIntroductoryPrice(product: self.monthlySubscription)
        let yearlyEligibility = await self.isEligibleForIntroductoryPrice(product: self.yearlySubscription)
        let specialYearlyEligibility = await self.isEligibleForIntroductoryPrice(product: self.specialYearlySubscription)
        self.discountable = monthlyEligibility && yearlyEligibility && specialYearlyEligibility
    }

    
}

```
