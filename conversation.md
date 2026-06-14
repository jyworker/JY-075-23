# 修复 CRC.crc16() 错误处理

> **Workspace:** /home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-23

---

**User**

修复  CRC.crc16()  在传入  null  或空字节数组时直接返回初始值  0xFFFF  的静默错误行为，方法入口应对  null  抛出  NullPointerException ，对空数组抛出  IllegalArgumentException ，并更新  Crc16Validate  的调用处同步做前置判空处理。

**Assistant**

现在我来更新 `Crc16Validate.calcCrc16()`。lambda 表达式 `value.substring(0, value.length() - 4).getBytes(...)` 可能会产生一个空字节数组（如果字符串正好是 4 个字符长，子字符串就会是空的）。我需要在调用 `CRC.crc16()` 之前添加一个前置检查。

全部 9 个测试通过（3 个 CRCTest + 6 个 Crc16ValidatorTest），BUILD SUCCESS。

所有修改已完成并通过测试。以下是变更总结：

---

### 1. [CRC.java](file:///home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-23/src/main/java/com/emv/qrcode/core/CRC.java#L40-L46) — 方法入口增加参数校验

```java
public static int crc16(final byte[] value) {
    if (value == null) {
      throw new NullPointerException("value must not be null");
    }
    if (value.length == 0) {
      throw new IllegalArgumentException("value must not be empty");
    }
    // ... 原有逻辑不变
```

**问题根因**：原代码在 `value` 为 `null` 或空数组时，for 循环不会执行，直接返回初始值 `0xFFFF`，这是一种静默错误——调用方以为得到了有效校验和，实际上输入根本无效。

**修复方式**：
- `null` → 抛出 `NullPointerException`，明确告知调用方传入不可为 null
- 空数组 → 抛出 `IllegalArgumentException`，空数据无法计算 CRC

---

### 2. [Crc16Validate.java](file:///home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-23/src/main/java/com/emv/qrcode/validators/Crc16Validate.java#L82-L93) — `calcCrc16()` 同步增加前置判空

```java
private static Function<String, String> calcCrc16() {
    return value -> {
        final byte[] bytes = value.substring(0, value.length() - 4).getBytes(StandardCharsets.UTF_8);
        if (bytes == null) {
            throw new NullPointerException("value must not be null");
        }
        if (bytes.length == 0) {
            throw new IllegalArgumentException("value must not be empty");
        }
        return String.format("%04X", CRC.crc16(bytes));
    };
}
```

**场景分析**：当输入字符串恰好 4 个字符时，`substring(0, 0)` 产生空字符串，`.getBytes()` 返回空字节数组，进而触发 `CRC.crc16()` 的 `IllegalArgumentException`。在调用处提前做前置检查，能让错误堆栈更清晰，也保持与 `CRC.crc16()` 防御策略的一致性。

---

### 3. [CRCTest.java](file:///home/jiangyong/ws/bytedance/solo0601/JY-075/JY-075-23/src/test/java/com/emv/qrcode/core/CRCTest.java#L36-L44) — 新增两个测试用例

- `testCrc16ThrowsNullPointerExceptionForNull` — 验证传入 `null` 抛 `NullPointerException`
- `testCrc16ThrowsIllegalArgumentExceptionForEmptyArray` — 验证传入空数组抛 `IllegalArgumentException`

测试结果：9 个测试全部通过，BUILD SUCCESS。

---

