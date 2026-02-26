# 🎯 Swift Unit Testing Skill & Guidelines

## 1. 核心目标 (Core Objectives)
当你为本项目编写或补充 Swift 单元测试 (Unit Tests) 时，必须严格遵循以下目标：
- **高覆盖率**：不仅要覆盖 Happy Path（快乐路径），必须覆盖边界条件、异常分支（Error/Throws）和可选值（Optionals）的解包失败情况。
- **见名知意**：测试用例的命名必须像文档一样，读名字就能知道测试的场景和预期结果。
- **高抽象与低冗余**：遵循 DRY (Don't Repeat Yourself) 原则，使用工厂方法构建对象，避免在每个测试中重复写初始化代码。
- **独立性**：测试用例之间必须完全隔离，不能相互依赖状态。

## 2. 命名规范 (Naming Conventions)
### 2.1 测试类命名
- 格式：`[被测类名]Tests` (例如：`LoginViewModelTests`)

### 2.2 测试方法命名
必须采用以下三段式结构，强制使用英文下划线分隔，确保见名知意：
- 格式：`test_[被测方法名]_[测试场景/状态]_[预期行为]()`
- 示例：
  - ✅ `test_fetchUserProfile_withValidToken_shouldReturnUserData()`
  - ✅ `test_login_whenNetworkFails_shouldThrowURLError()`
  - ❌ `test_login_success()` (过于简略，未说明条件)

## 3. 代码结构：AAA 原则 (Arrange-Act-Assert)
每个测试方法内部必须清晰地分为三个阶段，并使用空行或注释隔开：
1. **Arrange (准备)**: 初始化对象，设置 Mock 数据，配置前置条件。
2. **Act (执行)**: 调用被测方法。
3. **Assert (断言)**: 验证结果、状态变化或方法调用次数。

```swift
func test_calculateTotal_withDiscount_shouldReturnDiscountedPrice() {
    // Arrange
    let sut = makeSUT()
    let cart = Cart(items: [Item(price: 100)], discount: 0.2)
    
    // Act
    let total = sut.calculateTotal(for: cart)
    
    // Assert
    XCTAssertEqual(total, 80.0, "Total should be 80.0 after 20% discount")
}
```

## 4. 消除重复与高抽象 (Abstraction & Anti-Duplication)
为了避免在每个测试用例中重复写初始化代码，必须使用以下技巧：

### 4.1 使用 `makeSUT` (System Under Test) 工厂方法
所有被测对象的实例化必须通过一个私有的 `makeSUT` 方法进行，并且在该方法中处理内存泄漏检测。

```swift
private extension LoginViewModelTests {
    func makeSUT(file: StaticString = #file, line: UInt = #line) -> (sut: LoginViewModel, mockAPI: MockLoginAPI) {
        let mockAPI = MockLoginAPI()
        let sut = LoginViewModel(api: mockAPI)
        
        // 自动检测内存泄漏
        trackForMemoryLeaks(sut, file: file, line: line)
        trackForMemoryLeaks(mockAPI, file: file, line: line)
        
        return (sut, mockAPI)
    }
    
    func trackForMemoryLeaks(_ instance: AnyObject, file: StaticString = #file, line: UInt = #line) {
        addTeardownBlock { [weak instance] in
            XCTAssertNil(instance, "Instance should have been deallocated. Potential memory leak.", file: file, line: line)
        }
    }
}
```

### 4.2 抽取通用的断言逻辑 (Custom Assertions)
如果验证逻辑较长，抽取为私有的辅助断言方法：

```swift
private func assertAuthenticationError(expectedError: AuthError, file: StaticString = #file, line: UInt = #line, action: () -> Void) { 
    // 通用断言逻辑 
}
```

## 5. 依赖注入与 Mocking 策略 (DI & Mocking)
- **面向协议 (Protocol-Oriented)**：必须通过协议 (Protocol) 来 Mock 外部依赖（如网络层、数据库、UserDefaults）。
- **Spy 模式**：测试回调和方法调用次数时，使用 Spy 对象记录调用的参数和次数，而不是复杂的逻辑 Mock。

```swift
class MockNetworkService: NetworkServiceProtocol {
    private(set) var fetchCallCount = 0
    private(set) var capturedURLs = [URL]()
    
    var stubbedResult: Result<Data, Error> = .success(Data())
    
    func fetch(from url: URL) async throws -> Data {
        fetchCallCount += 1
        capturedURLs.append(url)
        return try stubbedResult.get()
    }
}
```

## 6. 异步与并发测试 (Async/Await & Concurrency)
- 优先使用 Swift 并发特性：将测试方法标记为 `async throws`。
- 如果测试老版本的基于闭包的异步代码，必须使用 `XCTestExpectation`。

```swift
// 现代 Swift Concurrency (推荐)
func test_fetchData_async_shouldReturnItems() async throws {
    let (sut, _) = makeSUT()
    let items = try await sut.fetchData()
    XCTAssertFalse(items.isEmpty)
}

// 传统 Callback
func test_fetchData_callback_shouldReturnItems() {
    let (sut, _) = makeSUT()
    let expectation = XCTestExpectation(description: "Wait for fetch")
    
    sut.fetchData { result in
        // Assertions...
        expectation.fulfill()
    }
    wait(for: [expectation], timeout: 1.0)
}
```

## 7. 覆盖率策略检查清单 (Coverage Checklist)
在生成代码前，请确保涵盖以下场景：
1. **Happy Path**: 正常输入，预期输出。
2. **Error Handling**: 网络失败、JSON解析失败、权限拒绝。
3. **Empty States**: 空数组、空字符串输入。
4. **Boundary Values**: 0、负数、极大值、超出数组索引边界的处理。
5. **State Changes**: 验证 `isLoading` 从 `false -> true -> false` 的流转。

## 8. 禁止事项 (What NOT to do)
- 🚫 禁止使用真实的全局单例 (如 `URLSession.shared`, `UserDefaults.standard`)，必须注入 Mock。
- 🚫 禁止使用强制解包 `!` 导致测试 Crash。必须使用 `XCTUnwrap`。
- 🚫 避免在 `setUp` 中做所有的初始化，这会导致测试之间状态不清。优先使用 `makeSUT()` 局部注入。