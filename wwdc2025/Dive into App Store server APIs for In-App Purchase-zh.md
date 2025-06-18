以下是文档“wwdc2025-249.txt”的完整中文翻译，遵循简洁、准确的原则，保留技术术语的原意并确保内容清晰易懂：

---

大家好，我是Riyaz，App Store服务器团队的工程师。  
在本场会议中，我们将深入探讨App Store服务器API在应用内购买（In-App Purchase）方面的功能。我将展示最新的更新如何优化和增强您的应用服务器职责。让我们首先了解您的应用服务器的一些关键职责。

在本场会议中，我将重点介绍三项关键职责：  
1. **管理应用内购买**：将交易数据与客户账户关联，以确保您的应用无缝提供内容和服务。  
2. **签名请求**：生成签名以授权您的服务器向App Store发送请求。  
3. **参与退款决策流程**：通过分享与购买相关的消费数据，帮助App Store做出明智的退款决策。  

这些是您的应用服务器管理的众多关键职责之一。接下来，我将介绍App Store服务器API的更新如何增强这些职责。内容很多，让我们直接开始！  

首先，我将介绍交易标识符的更新，帮助您更好地管理应用内购买。  
然后，我会讲解生成签名的改进，简化服务器上的签名请求。  
最后，我会介绍简化退款流程参与的增强功能。  

### 管理应用内购买  
管理应用内购买始于在您的系统上有效处理客户账户。通常，您会为每个客户分配一个唯一的账户ID，建立客户账户与App Store交易的明确关联。这种关联对于提供正确内容或个性化用户体验至关重要。  

App Store通过三种关键数据结构提供应用内购买数据：**AppTransaction**、**JWSTransaction**和**JWSRenewalInfo**。  
我首先重点介绍**JWSTransaction**，稍后再讨论其他类型。  

当客户进行应用内购买时，App Store会提供一个签名交易对象。在服务器上，**JWSTransaction**表示这个签名交易，您可以使用App Store服务器库（App Store Server Library）进行验证和解码。  

以下是一个解码后的**signedTransactionInfo**示例：  
- 前几个字段包含与应用和产品类型相关的重要信息。  
- 接下来是购买的元数据，包括数量、价格和货币。  
- 如果客户兑换了优惠，**JWSTransaction**会包含**offerType**、**offerIdentifier**和**offerDiscountType**字段。  
- 我们最近新增了**offerPeriod**字段，使用ISO 8601格式表示优惠的持续时间。此字段也包含在**JWSRenewalInfo**中，告知您下次订阅续订时适用的优惠时长。  

接下来是交易标识符：  
- **transactionId**：交易的唯一标识符，例如应用内购买、恢复或订阅续订。例如，它可以表示订阅续订的交易ID。  
- **originalTransactionId**：原始购买的交易ID，有助于准确识别自动续订订阅，因为它在订阅续订中保持一致。  
- **appAccountToken**：您在应用内使用StoreKit设置的UUID，用于在客户进行应用内购买时关联到您的系统账户。  

为便于获取订阅续订的**appAccountToken**，我们已将其包含在**JWSRenewalInfo**中，帮助您无缝关联客户账户与其最新订阅续订交易。  

以下是其工作原理：  
1. 您在服务器上生成一个UUID，并将其与客户账户关联。  
2. 服务器将此值传递给使用StoreKit构建的应用，在应用内购买时设置**appAccountToken**。  
3. 我们建议为同一客户账户的所有应用内购买使用相同的**appAccountToken**值。  
4. App Store服务器会在**JWSTransaction**和**JWSRenewalInfo**中返回相同的**appAccountToken**值。  

过去，您只能在应用内购买时设置**appAccountToken**。但客户也可能在应用外购买，例如兑换优惠码或在App Store进行促销购买。为此，我们推出了新的**设置App Account Token端点**，允许您为交易设置或更新**appAccountToken**。  

此端点适用于所有产品类型，包括：  
- 一次性购买（如消耗品、非消耗品和非续订订阅）。  
- 自动续订订阅的最新购买（设置的**appAccountToken**会延续到未来的续订，包括升级或降级）。  

此端点特别适用于客户在应用外完成交易（如兑换优惠码）的情况，过去无法为这类购买设置**appAccountToken**，因为未涉及StoreKit。此外，您还可以修复账户关联不一致问题，例如账户所有权变更。  

以下是使用**设置App Account Token端点**的示例：  
- 在路径中提供**originalTransactionId**，标识一次性购买交易或要设置**appAccountToken**的订阅。  
- 在请求体中，设置所需的UUID值作为**appAccountToken**。  
- 新设置的**appAccountToken**会覆盖该交易的任何先前值。  

**appAccountToken**与**transactionId**和**originalTransactionId**一起，为关联客户账户与App Store交易提供了便利：  
- **transactionId**：用于标识特定购买事件。  
- **originalTransactionId**：用于管理自动续订订阅的生命周期和检查订阅状态。  
- **appAccountToken**：用于将客户账户信息与应用内购买关联。如果您的应用支持家庭共享，请注意，**appAccountToken**不适用于家庭共享交易。  

**transactionId**和**originalTransactionId**存在于**JWSTransaction**和**JWSRenewalInfo**中。**appAccountToken**仅在您通过StoreKit或新端点设置时可用。  

当客户下载您的应用时，**AppTransaction**对象通过StoreKit提供，包含重要的应用下载信息。**transactionId**、**originalTransactionId**和**appAccountToken**特定于应用内购买，因此不包含在**AppTransaction**对象中。  

但有时需要唯一标识应用下载，例如在应用安装时解锁内容。为此，我们在**AppTransaction**对象中新增了**appTransactionId**字段。  
- **appTransactionId**是每个Apple账户对每个应用的全局唯一标识符。  
- 对于家庭共享，每个家庭成员有唯一的**appTransactionId**。  
- **appTransactionId**在重新下载、退款、重新购买或商店变更时保持不变。  
- **appTransactionId**也包含在客户在您的应用内的所有应用内购买中，增加关联客户账户的灵活性。  

以下是使用**appTransactionId**关联客户账户的示例：  
1. 客户下载应用时，将您的系统中的客户账户与**AppTransaction**对象中的**appTransactionId**关联。  
2. 您可以通过应用发送**AppTransaction**对象到服务器。  
3. 客户进行应用内购买时，**JWSTransaction**和**JWSRenewalInfo**包含相同的**appTransactionId**，可重复使用以关联所有App Store交易对象。  

**appTransactionId**可用于识别同一客户账户的多个订阅购买。例如，假设您的应用支持两种订阅产品：月度体育新闻和年度直播比赛订阅。您可以通过共享的**appTransactionId**判断客户是否拥有两种订阅。  

此外，**appTransactionId**可用于热门App Store服务器API端点，如获取交易历史、所有订阅状态等。  

我们很高兴推出新的**获取App Transaction Info端点**，首次允许您直接在服务器上获取应用下载信息，而无需依赖设备。您可以使用**AppTransaction**对象检查应用版本、平台和环境等信息，以了解应用在业务模型变更中的表现。  

请注意，此端点不返回设备相关信息（如设备验证），需继续从应用获取**AppTransaction**对象。  

以下是使用**获取App Transaction Info端点**的示例：  
- 在路径中提供任意交易标识符（如**originalTransactionId**、**transactionId**或**appTransactionId**）。  
- 响应包含**JWS signedAppTransactionInfo**，可使用App Store服务器库解码。  
- 此端点将于今年晚些时候推出。  

以下是**appTransactionId**与其他交易标识符的比较：  
- App Store生成**appTransactionId**、**transactionId**和**originalTransactionId**。  
- **appTransactionId**用于唯一标识应用下载，并关联客户后续购买。  
- **appTransactionId**在所有App Store交易对象中一致，且支持家庭共享交易。  
- 我们推荐使用**appTransactionId**，因为它简化了应用内购买管理和客户账户关联。  

### 签名请求的改进  
签名请求的第一步是在服务器上创建JWS（JSON Web Signature）字符串，使用从App Store Connect下载的私钥签名。然后将签名字符串发送到使用StoreKit构建的应用，调用需要签名的函数，StoreKit再将签名字符串发送到App Store服务器。  

过去，签名格式因用例而异。现在，我们将所有用例的签名请求统一为JWS签名格式，适用于包括生成促销优惠签名在内的StoreKit调用。  

我们还为入门优惠（introductory offers）引入了新的JWS签名功能，允许您为每个交易和用户设置自定义资格，控制客户兑换的入门优惠数量。  

您还可以使用高级商务API发送签名的JWS应用内请求。更多信息请参阅开发者文档。  

以下是使用App Store服务器库创建JWS促销优惠签名的示例（以Java为例）：  
1. 使用私钥、**keyId**、**issuerId**和应用的**bundleId**实例化**PromotionalOfferV2SignatureCreator**类（这些值可在App Store Connect找到）。  
2. 指定**transactionId**（可使用客户的任意交易标识符，包括**appTransactionId**，或跳过此字段以不限制特定客户）。  
3. 提供**productId**和**offerId**（已在App Store Connect预配置）。  
4. 将**productId**、**offerId**和**transactionId**传递给**createSignature**函数。  

新签名比旧版本更简单，输入字段更少，统一JWS签名格式简化了多种用例的签名请求。  

### 参与退款决策流程  
即使提供了优惠，客户仍可能因账单问题或意外购买请求退款。以下是参与退款决策的好处和方式：  
参与退款决策让您利用对客户产品消费的了解（例如管理消耗品余额），提高退款后的客户满意度。  

当客户请求退款时，App Store会向您的服务器发送**CONSUMPTION_REQUEST**通知。您可以使用**发送消费信息V2端点**回应此通知，提供消费数据以协助退款决策。  

我们推出了改进的**发送消费信息V2端点**，特点包括：  
- 相比旧版本，输入字段从12个减少到5个，集成更简单。  
- 支持按比例退款（prorated refund）选项，适用于部分消费产品。  
- 扩展支持所有产品类型（包括非消耗品和非续订订阅）。  

若您已使用V1端点，请注意它已被弃用，但仍接受请求。我们推荐使用最新的V2端点与App Store服务器库。  

以下是使用**发送消费信息V2端点**的示例：  
- 假设客户为消耗品请求退款，服务器收到**CONSUMPTION_REQUEST**通知。  
- 在路径中提供通知中的**transactionId**。  
- 端点支持5个字段（3个必填，2个可选）：  
  - **customerConsented**：若客户同意发送消费数据，设为true；否则不回应通知（设为false将拒绝请求）。  
  - **sampleContentProvided**：指定购买前是否提供样品内容。  
  - **deliveryStatus**：若内容成功交付，设为DELIVERED；否则使用适当的UNDELIVERED状态。  
  - **refundPreference**（可选）：可选择GRANT_PRORATED（按比例退款）、全额退款或无退款。  
  - **consumptionPercentage**（可选）：以毫百分比表示产品消费，例如25000表示25%消费。  

此端点将于今年晚些时候推出。  

### 按比例退款选项  
在客户部分消费产品的情况下，建议提供按比例退款偏好，指定**consumptionPercentage**，帮助App Store确定适当退款金额。  
- 对于消耗品、非消耗品和非续订订阅，需提供**consumptionPercentage**。  
- 对于自动续订订阅，App Store根据剩余订阅时间自动计算**consumptionPercentage**。  

您提交的消费数据帮助App Store决定全额退款、按比例退款或拒绝退款。您会收到**REFUND**或**REFUND_DECLINED**通知：  
- **REFUND_DECLINED**：无需操作。  
- **REFUND**：需采取适当行动。  

**REFUND**通知示例：  
- 新增的**refundPercentage**字段显示退款比例（例如75%）。  
- **revocationType**字段显示撤销类型（REFUND_FULL、REFUND_PRORATED或FAMILY_REVOKE）。  
- 对于消耗品退款，可能值为REFUND_FULL或REFUND_PRORATED。  

处理按比例退款时：  
- 对于消耗品、非消耗品或非续订订阅，使用**refundPercentage**计算并撤销相应比例的内容（例如减少虚拟货币余额）。  
- 对于自动续订订阅，按全额退款方式处理，检查当前订阅状态并采取行动。  

### 总结  
我们介绍了：  
1. 各种交易标识符，及新的**appTransactionId**作为一站式标识符。  
2. 统一的JWS签名格式。  
3. 参与退款决策的简化流程。  

### 下一步  
- 如果您已为开源App Store服务器库做出贡献，感谢您！否则，请访问我们的GitHub页面了解如何参与支持App Store社区。  
- 请通过Feedback Assistant提交功能请求和反馈。  
- 了解更多StoreKit更新，请观看WWDC25“StoreKit和应用内购买新功能”会议。  
- 查看WWDC24“探索App Store服务器API”会议，了解更多详情。  

感谢您的参与，下次见！

---

以上翻译力求准确传达原文的技术细节和语义，同时使用简洁的中文表达，符合技术文档的风格。如果需要进一步调整或补充，请告知！