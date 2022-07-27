# IAPStoreKitAPI
## 配置工程

1. 点击 scheme 打开 Scheme menu 并选择 Edit Scheme
2. 在scheme 编辑器中选择 Run action
3. 设置中点击 Options
4. StoreKit Configuration 选择 Products.storekit 配置

## 基础知识

App 内购买项目能让用户在您的 App 中安全地购买***内容、功能或服务***。每个 App 最多可以创建 10,000 个 App 内购买项目。App 内购买项目共有四种类型：消耗型、非消耗型、自动续期订阅和非续期订阅。

<aside> 📌 App中购买商品不需要使用App内购买项目，比如淘宝、京东等电商类App。

苹果提供的技术框架是 StoreKit，里面最主要的就是 In-App Purchase，俗称 iap，也就是App内购买项目。

<aside> ♨️ 用户购买商品，我们需要提供订单和支付两大系统来支持。不同于微信和支付包支付只提供支付能力，iap同时提供了订单和支付两种系统能力，这在没有服务端的App场景下非常便利

首先来认识iap的四种类型。

![](/img/type.png)

我们可以通过 Product 的 type 来判断当前商品属于哪种类型：

```swift
switch product.type {
case .consumable:
print("消耗型项目")
case .nonConsumable:
print("非消耗型项目")
case .autoRenewable:
print("自动续费订阅")
case .nonRenewable:
print("非续费订阅")
default:
//Ignore this product.
print("Unknown product")
}
```

在我们配置好App 内购买项目，要在App 中支持购买商品，需要实现以下功能：

- updates 交易监听器，监听交易状态的变更。
- products(for:) 从 App Store 请求获取到产品信息。
- purchase(options:) 购买商品
- currentEntitlements 交易功能的一个属性，包含用户购买的内容和服务
- VerificationResult  验证交易的签名和订阅状态信息

```swift
// 在 App lanch 后就添加交易监听器，以免丢失交易
updateListenerTask = listenForTransactions()

func listenForTransactions() -> Task<Void, Error> {
        return Task.detached {
            //Iterate through any transactions that don't come from a direct call to `purchase()`.
            for await result in Transaction.updates {
                do {
                    let transaction = try self.checkVerified(result)

                    //Deliver products to the user.
                    await self.updateCustomerProductStatus()

                    //Always finish a transaction.
                    await transaction.finish()
                } catch {
                    //StoreKit has a transaction that fails verification. Don't deliver content to the user.
                    print("Transaction failed verification")
                }
            }
        }
    }

// 在app销毁时取消监听
deinit {
    updateListenerTask?.cancel()
}
```

```swift
func requestProducts() async {
  do {
    // 使用商品的identifiers请求商品信息
    let storeProducts = try await Proudct.products(for: productIdToEmoji.keys)
    for product in storeProducts {
      switch product.type {
        case .consumable:
        print("消耗型项目")
        case .nonConsumable:
				print("非消耗型项目")
				case .autoRenewable:
				print("自动续费订阅")
				case .nonRenewable:
				print("非续费订阅")
				default:
				//Ignore this product.
				print("Unknown product")
      }
    }
  } catch {
    print("Failed product request from the App Store server:\(error)")
  }
}
```

```swift
func purchase(_ product: Product) async throws -> Transaction? {
    // 开始购买用户选择的产品
	 let result = try await product.purchase()
   
   switch result {
	 case .success(let verification):
      //验证交易信息是否正确，如果不正确，这个方法会抛出验证的异常信息
      let transaction = try checkVerified(verification)
      //交易信息验证通过，交付内容给用户
			await updateCustomerProductStatus()
		  //总是完成一笔交易
			await transaction.finish()
      return transaction
	 }
   case .userCancelled, .pending:
			return nil
   default:
      return nil
}
```

```swift
func updateCustomerProductStates() async {
		 // 循环所有用户已购买的商品
		 for await result in Transaction.currentEntitlements {
				 do {
		         //验证交易信息是否正确，如果不正确，这个方法会抛出验证的异常信息
				     let transaction = try checkVerified(verification)
				 } catch {
						print()
				 }
     }
}
```

```swift
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
```

以上代码环境是 iOS15+、iPadOS15+、Xcode13.4+ 

[StoreKit](https://developer.apple.com/documentation/storekit)

[StoreKit-IAP](https://developer.apple.com/documentation/storekit/in-app_purchase/implementing_a_store_in_your_app_using_the_storekit_api)

