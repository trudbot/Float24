
> 你从xx大学毕业后，顺利考上了某校研究生。你的导师正在进行人工智能方面的研究，
>
> 面对庞大的数据集以及数据。你的导师发现，传统的float占据4个字节，数据范围和精度都能满足要求
>
> ，但是占据内存空间过大，实验室机器内存无法达到训练要求； 若采用float16这种2字节的，虽然内存
>
> 使用量降低了一半，但是精度和范围又达不到，很多数据超过了范围存入不了。
>
> 因此， 你的导师希望你能够参照系统已有的float类型，何double类型，设计一个3个字节的浮点
>
> 数，请详细设计该float类型的符号位，阶码，尾码所占字节数； 请详细设计一个浮点数，转换成该种
>
> 设计下的二进制转换方法。请详细设计0，NAN，INF等特殊数据的存储方式。
>
> 依据你的设计，利用c，c++完成Float24这个类，并实现一个数据例如3.55这个数据存储到该
>
> Float24变量， Float24 a; a.set(3.55); 以及从二进制表示下，取出该值， 例如a.get()， 得到3.55这个
>
> 数。并实现上述Float24 数据类型的，正确的加减乘除运算，其中运算正确，存储正确。

## 数据结构设计

参照IEE-754标准, 进行24位浮点数数据结构约定:

![未命名文件 (1)](https://trudbot-md-img.oss-cn-shanghai.aliyuncs.com/202406261636956.png)

* 符号位为`1`位
* 指数位为`7`位, 指数偏移量$bias = 2^{7 - 1} - 1 = 63$, 指数表示范围为$[-63, 64]$
* 尾数位为`16`位

### 约定

符号位、指数位、尾数位的含义均与IEE-754相同， 浮点数特殊情况如下: 

|    形式    |   指数值    | 尾数位  |               符号位               |
| :--------: | :---------: | :-----: | :--------------------------------: |
|     零     |     -63     |  全为0  | 为0时为`+0`， 1时为`-0`， 两者等价 |
|  规约形式  | $[-62, 63]$ |    /    |                 /                  |
| 非规约形式 |     -63     | 非全为0 |                 /                  |
|    无穷    |     64      |  全为0  | 为0时为$+\infty$， 1时为$-\infty$  |
|    NAN     |     64      | 非全为0 |                 /                  |

### 指数位为什么设置为7位

1. 参考IEE754中对16位、32位等等浮点类型的指数位位数定义， 综合 **范围**和**精度**考虑

| 浮点数长度 | 指数位位数 |
| ---------- | :--------: |
| 16         |     5      |
| 32         |     8      |
| 64         |     11     |
| 128        |     15     |

使用7位的指数位， 表示范围:  $\pm (2 - \frac{1}{2^{16}}) * 2^{63}$ ,  精度: $len(2^{16})$ = 5位

2. 实现上略微带来便利

1位的符号位和7位的指数位刚好占满一个字节， 所以操作符号和指数时只需要操作第一个字节， 操作尾数时只需要操作后面两个字节。

3. 参考工业界实现

[Radeon R300](https://en.wikipedia.org/wiki/Radeon_R300) 和 [R420](https://en.wikipedia.org/wiki/Radeon_R420) GPU 使用“fp24”浮点格式，具有 7 位指数和 16 位（+1 隐式）尾数

## 实现

### 将十进制形式的浮点数转换为3字节浮点数

1. **解析字符串**：将输入的字符串分解为符号位、整数部分和小数部分。
2. **转换为二进制**：将整数部分和小数部分分别转换为二进制表示。
3. **规格化**：将转换后的二进制数规格化，使其形式为1.xxxx，并调整指数。
4. **编码**：根据Float24的格式，将规格化后的数编码为符号位、指数位和尾数位。

```cpp
static void stringToFloat24(std::string& s, uint8_t bytes[3], int expBits, int mantissaBits, int bias) {

    // not a number
    if (!isDecimalFloat(s)) {
        bytes[0] = 0x7F;
        bytes[1] = bytes[2] = 0xFF;
        return;
    }

    /**
     * 读取符号位
     * 如果存在符号， 将字符串符号去掉
     */
    if (isdigit(s[0])) {
        bytes[0] = 0x00;
    } else {
        bytes[0] = s[0] == '+' ? 0x00 : 0x80;
        s = s.substr(1);
    }

    /**
     * 将字符串分解为整数部分和小数部分
     */
    std::string integerPart, fractionalPart;
    splitAtDecimal(s, integerPart, fractionalPart);
    // cout << "integerPart: " << integerPart << endl;
    // cout << "fractionalPart: " << fractionalPart << endl;

    // 将整数部分和小数部分转换为二进制
    std::string integerBinary = integerToBinary(std::stoi(integerPart));
    std::string fractionalBinary;
    if (fractionalPart.size() == 0) {
        fractionalBinary = "";
    } else {
        fractionalBinary = fractionToBinary(std::stod("0." + fractionalPart), 16);
    }
    // cout << "integerBinary: " << integerBinary << endl;
    // cout << "fractionalBinary: " << fractionalBinary << endl;

    // 规范化 --> 1.xxx * 2^exp
    // 尾数部分
    std::string mantissa = integerBinary.substr(1) + fractionalBinary;
    // 指数值
    int exp = integerBinary.size() - 1 + bias;
    // cout << "exp: " << exp << endl;

    // 指数溢出, 设置为无穷
    if (exp > 127) {
        // 指数位置1
        bytes[0] = bytes[0] | 0x7F;
        // 尾数位置0
        bytes[1] = 0x00;
        bytes[2] = 0x00;
        return;
    }

    // 指数转化为二进制填入指数位中
    bytes[0] |= (exp & 0b01111111);

    // 将尾数填入尾数位中
    for (int i = 0; i < mantissa.size() && i < 16; ++i) {
        if (mantissa[i] == '1') {
            if (i < 8) {
                // 填入 bytes[1]
                bytes[1] |= (1 << (7 - i));
            } else {
                // 填入 bytes[2]
                bytes[2] |= (1 << (15 - i));
            }
        }
    }
}
```



### 浮点数运算

#### 乘法

浮点数的乘法分为以下几个步骤：

- 计算符号位：通过异或操作计算符号位，若两个操作数符号位相同，则结果符号位为0，否则结果符号为1
- 计算原始尾数：两个操作数的尾数相乘，得到原始尾数
- 计算原始指数：将两个操作数的指数相加，得到原始指数
- 规格化与舍入：对原始尾数和原始指数进行规格化，获得结果的指数，再对尾数进行舍入，获得结果的尾数

[![img](https://trudbot-md-img.oss-cn-shanghai.aliyuncs.com/202406261922915.png)](https://www.yuejianzun.xyz/2019/05/28/浮点数处理/mul_flow.png)

```cpp
static void multiply(const uint8_t a[3], const uint8_t b[3], uint8_t result[3]) {
    // 提取符号位
    bool signA = a[0] & 0x80;
    bool signB = b[0] & 0x80;

    // 提取指数
    int exponentA = (a[0] & 0x7F) - 63;
    int exponentB = (b[0] & 0x7F) - 63;

    // 提取尾数
    uint32_t mantissaA = (1 << 16) | ((a[1] << 8) | a[2]);
    uint32_t mantissaB = (1 << 16) | ((b[1] << 8) | b[2]);

    // 计算结果的指数和尾数
    int resultExponent = exponentA + exponentB;
    uint64_t mantissaResult = static_cast<uint64_t>(mantissaA) * mantissaB;

    // 归一化处理
    // 尾数规格化，确保尾数的最高位为1，并调整指数
    if (mantissaResult & (1ULL << (32 + 1))) { // 如果最高位为1，表示结果是33位
        mantissaResult >>= 1;
        ++resultExponent;
    }

    // 去掉额外的位数，保留规定的尾数位数
    mantissaResult >>= 16;

    // 处理溢出, 设置为无穷
    if (resultExponent > 63) {
        result[0] = signA ^ signB ? 0x80 : 0xFF;
        result[1] = 0x00;
        result[2] = 0x00;
        return;
    }

    // 处理下溢
    if (resultExponent < -63) {
        result[0] = signA ^ signB ? 0x80 : 0x00;
        result[1] = 0x00;
        result[2] = 0x00;
        return;
    }

    // 编码结果
    result[0] = (signA ^ signB ? 0x80 : 0x00) | ((resultExponent + 63) & 0x7F);
    result[1] = (mantissaResult >> 8) & 0xFF;
    result[2] = mantissaResult & 0xFF;
}

```

#### 加法

1. **比较指数位**：找出指数较大的数，并将较小的数的尾数右移，使得两个数的指数相同。
2. **处理尾数**：根据符号位，执行尾数的加法或减法操作。
3. **规格化结果**：可能需要左移或右移尾数，以确保其格式正确，并相应调整指数。
4. **溢出和下溢处理**：检查结果是否溢出或下溢。

```cpp
/**
 * 浮点数加法
 * @param a 加数
 * @param b 加数
 * @param result 结果
 */
static void add(uint8_t a[3], uint8_t b[3], uint8_t result[3]) {
    // 提取符号位
    bool signA = a[0] & 0x80;
    bool signB = b[0] & 0x80;

    // 提取指数
    int exponentA = (a[0] & 0x7F) - 63;
    int exponentB = (b[0] & 0x7F) - 63;

    // 提取并解码尾数位，加上隐含的前导1, 共17位
    uint32_t mantissaA = (1 << 16) | (a[1] << 8) | a[2];
    uint32_t mantissaB = (1 << 16) | ((b[1] << 8) | b[2]);

    // 对齐指数
    if (exponentA > exponentB) {
        mantissaB >>= (exponentA - exponentB);
        exponentB = exponentA;
    } else if (exponentB > exponentA) {
        mantissaA >>= (exponentB - exponentA);
        exponentA = exponentB;
    }

    uint32_t mantissaResult;
    int resultExponent = exponentA;

    // 加法或减法
    if (signA == signB) {
        mantissaResult = mantissaA + mantissaB;
        if (mantissaResult & (1 << 17)) { // 溢出，需要右移一位
            mantissaResult >>= 1;
            ++resultExponent;
        }
    } else {
        if (mantissaA > mantissaB) {
            mantissaResult = mantissaA - mantissaB;
            signB = signA;
        } else {
            mantissaResult = mantissaB - mantissaA;
            signA = signB;
        }
        // 左移直到尾数位为17位
        while ((mantissaResult & (1 << 16)) == 0 && mantissaResult != 0) { // 归一化
            mantissaResult <<= 1;
            --resultExponent;
        }
    }

    // 处理溢出, 设置为无穷
    if (resultExponent > 63) {
        result[0] = signA ? 0xFF : 0x7F;
        result[1] = 0x00;
        result[2] = 0x00;
        return;
    }

    // 处理下溢
    if (resultExponent < -63) {
        result[0] = signA ? 0x80 : 0x00;
        result[1] = 0x00;
        result[2] = 0x00;
        return;
    }

    // 编码结果
    result[0] = (signA ? 0x80 : 0x00) | ((resultExponent + 63) & 0x7F);
    result[1] = (mantissaResult >> 8) & 0xFF;
    result[2] = mantissaResult & 0xFF;
}
```

#### 减法

1. 将被减数符号位取反
2. 减数 与 被减数 进行加法

```cpp
Float24 operator-(Float24 & b) const {
    uint8_t result[3] = {0, 0, 0};
    // 将b符号位取反
    uint8_t* bBytes = b.getBytes();
    uint8_t bCopy[3] = {static_cast<uint8_t>(bBytes[0] ^ 0x80), bBytes[1], bBytes[2]};
    Float24::add(bytes, bCopy, result);
    return Float24(result);
}
```

#### 除法

**提取符号位、指数和尾数**：与乘法类似，先提取符号位、指数和尾数。

**计算结果的符号位**：使用符号位异或计算结果的符号位。

**计算结果的指数**：被除数的指数减去除数的指数，并加上偏移量。

**计算结果的尾数**：被除数的尾数除以除数的尾数，并进行规格化。

**处理溢出和下溢**：如果结果的指数超出表示范围，设置为无穷或零。

**将结果编码到 `result` 数组中**。

```cpp
static void divide(const uint8_t a[3], const uint8_t b[3], uint8_t result[3]) {
        // 提取符号位
        bool signA = a[0] & 0x80;
        bool signB = b[0] & 0x80;

        // 提取指数
        int exponentA = (a[0] & 0x7F) - 63;
        int exponentB = (b[0] & 0x7F) - 63;

        // 提取尾数并加入隐含的前导1
        uint32_t mantissaA = (1 << 16) | ((a[1] << 8) | a[2]);
        uint32_t mantissaB = (1 << 16) | ((b[1] << 8) | b[2]);

        // 计算结果的符号位
        bool resultSign = signA ^ signB;

        // 计算结果的指数
        int resultExponent = exponentA - exponentB;
        dev && cout << endl << "规格化前exp " << resultExponent << endl;
        // 计算结果的尾数
        // 1.xx * 2^(16 + 17) / 1.yy * 2^16 = 1.zz * 2^17
        // 除数左移17位是防止结果有效位数小于17位
        uint64_t mantissaResult = ((static_cast<uint64_t>(mantissaA)) << 17) / mantissaB;
        resultExponent -= 1;
        dev && cout << "mantissaResult " << mantissaResult << endl;
        // 将尾数右移，直到为17位
        while (mantissaResult && mantissaResult >= (1ULL << 17)) {
            resultExponent ++;
            mantissaResult >>= 1;
        }
        dev && cout << endl << "resultExp " << resultExponent << endl;
        // 处理溢出, 设置为无穷
        if (resultExponent > 63) {
            result[0] = resultSign ? 0xFF : 0x7F;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 处理下溢
        if (resultExponent < -63) {
            result[0] = resultSign ? 0x80 : 0x00;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 编码结果
        result[0] = (resultSign ? 0x80 : 0x00) | ((resultExponent + 63) & 0x7F);
        result[1] = (mantissaResult >> 8) & 0xFF; // 取高8位
        result[2] = mantissaResult & 0xFF;        // 取次高8位
    }
```

## 完整代码

```cpp
#include <cmath>
#include <cstdint>
#include <iostream>
#include <stdexcept>
#include <regex>
#include <string>
using namespace std;

const bool dev = false;

bool isDecimalFloat(const std::string& str) {
    // 定义正则表达式
    std::regex floatRegex("[+-]?(\\d+(\\.\\d*)?|\\.\\d+)");

    // 使用正则表达式匹配
    return std::regex_match(str, floatRegex);
}

void splitAtDecimal(const std::string& str, std::string& integerPart, std::string& fractionalPart) {
    // 查找小数点的位置
    size_t decimalPos = str.find('.');

    if (decimalPos != std::string::npos) {
        // 分割字符串
        integerPart = str.substr(0, decimalPos);
        fractionalPart = str.substr(decimalPos + 1);
    } else {
        // 如果没有小数点，整个字符串就是整数部分
        integerPart = str;
        fractionalPart = "";
    }
}

// 将整数部分转换为二进制
std::string integerToBinary(int num) {
    std::string result;
    while (num > 0) {
        result = (num % 2 == 0 ? "0" : "1") + result;
        num /= 2;
    }
    return result.empty() ? "0" : result; // 处理num为0的情况
}

/**
 * 将小数转换为二进制
 * @param num 小数
 * @param precision 精度
 */
std::string fractionToBinary(double num, int precision) {
    std::string result;
    while (num > 0 && result.length() < precision) {
        num *= 2;
        if (num >= 1) {
            result += "1";
            num -= 1;
        } else {
            result += "0";
        }
    }
    return result;
}

class Float24 {
private:
    uint8_t bytes[3];
    int expBits =  7;
    int mantissaBits = 16;
    int bias = 63;
public:
    Float24() {
        bytes[0] = bytes[1] = bytes[2] = 0x00;
    }
    Float24(std::string& s) {
        stringToFloat24(s, bytes, expBits, mantissaBits, bias);
    }

    Float24(const std::string& s) {
        string sCopy = s;
        stringToFloat24(sCopy, bytes, expBits, mantissaBits, bias);
    }

    Float24(uint8_t bytes[3]) {
        this->bytes[0] = bytes[0];
        this->bytes[1] = bytes[1];
        this->bytes[2] = bytes[2];
    }

    bool isNan() {
        return (bytes[0] == 0x7F || bytes[0] == 0xFF) && bytes[1] == 0xFF && bytes[2] == 0xFF;
    }

    bool isInf() {
        return (bytes[0] == 0x7F || bytes[0] == 0xFF) && bytes[1] == 0x00 && bytes[2] == 0x00;
    }

    bool isZero() {
        return (bytes[0] == 0x00 || bytes[0] == 0x80) && bytes[1] == 0x00 && bytes[2] == 0x00;
    }

    uint8_t* getBytes() {
        return bytes;
    }

    // 以 0/1 形式打印 bytes
    void printBytes() {
        for (int i = 0; i < 3; ++i) { // 假设bytes数组有3个元素
            if (i == 1) std::cout << " ";
            for (int j = 7; j >= 0; --j) { // 遍历每个字节的每一位
                std::cout << ((this->bytes[i] & (1 << j)) ? '1' : '0');
                if (i == 0 && j == 7) std::cout << " ";
            }
        }
        std::cout << std::endl; // 打印完所有字节后换行
    }

    static void stringToFloat24(std::string& s, uint8_t bytes[3], int expBits, int mantissaBits, int bias) {

        // not a number
        if (!isDecimalFloat(s)) {
            bytes[0] = 0x7F;
            bytes[1] = bytes[2] = 0xFF;
            return;
        }

        /**
         * 读取符号位
         * 如果存在符号， 将字符串符号去掉
         */
        if (isdigit(s[0])) {
            bytes[0] = 0x00;
        } else {
            bytes[0] = s[0] == '+' ? 0x00 : 0x80;
            s = s.substr(1);
        }

        /**
         * 将字符串分解为整数部分和小数部分
         */
        std::string integerPart, fractionalPart;
        splitAtDecimal(s, integerPart, fractionalPart);
        dev && cout << "integerPart: " << integerPart << endl;
        dev && cout << "fractionalPart: " << fractionalPart << endl;

        // 将整数部分和小数部分转换为二进制
        std::string integerBinary = integerToBinary(std::stoi(integerPart));
        std::string fractionalBinary;
        if (fractionalPart.size() == 0) {
            fractionalBinary = "";
        } else {
            fractionalBinary = fractionToBinary(std::stod("0." + fractionalPart), 16);
        }
        dev && cout << "integerBinary: " << integerBinary << endl;
        dev && cout << "fractionalBinary: " << fractionalBinary << endl;

        // 规范化 --> 1.xxx * 2^exp
        std::string normalizedBinary = integerBinary + fractionalBinary;
        size_t firstOnePos = normalizedBinary.find('1');

        // 如果找不到1，设为0
        if (firstOnePos == std::string::npos) {
            bytes[0] |= 0x80;
            bytes[1] = bytes[2] = 0x00;
            return;
        }

        // 提取尾数和指数
        int exp = integerBinary.length() - firstOnePos - 1 + bias;
        std::string mantissa = normalizedBinary.substr(firstOnePos + 1);
        dev && cout << "exp: " << exp << endl;
        dev && cout << "mantissa: " << mantissa << endl;
        
        // 指数溢出, 设置为无穷
        if (exp > 127) {
            // 指数位置1
            bytes[0] = bytes[0] | 0x7F;
            // 尾数位置0
            bytes[1] = 0x00;
            bytes[2] = 0x00;
            return;
        }

        // 指数转化为二进制填入指数位中
        bytes[0] |= (exp & 0b01111111);

        // 将尾数填入尾数位中
        bytes[1] = bytes[2] = 0x00;
        for (int i = 0; i < mantissa.size() && i < 16; ++i) {
            if (mantissa[i] == '1') {
                if (i < 8) {
                    // 填入 bytes[1]
                    bytes[1] |= (1 << (7 - i));
                } else {
                    // 填入 bytes[2]
                    bytes[2] |= (1 << (15 - i));
                }
            }
        }
    }

    /**
     * 浮点数加法
     * @param a 加数
     * @param b 加数
     * @param result 结果
     */
    static void add(const uint8_t a[3], const uint8_t b[3], uint8_t result[3]) {
        // 提取符号位
        bool signA = a[0] & 0x80;
        bool signB = b[0] & 0x80;

        // 提取指数
        int exponentA = (a[0] & 0x7F) - 63;
        int exponentB = (b[0] & 0x7F) - 63;

        // 提取并解码尾数位，加上隐含的前导1, 共17位
        uint32_t mantissaA = (1 << 16) | ((a[1] << 8) | a[2]);
        uint32_t mantissaB = (1 << 16) | ((b[1] << 8) | b[2]);

        // 对齐指数
        if (exponentA > exponentB) {
            mantissaB >>= (exponentA - exponentB);
            exponentB = exponentA;
        } else if (exponentB > exponentA) {
            mantissaA >>= (exponentB - exponentA);
            exponentA = exponentB;
        }

        uint32_t mantissaResult;
        int resultExponent = exponentA;

        // 加法或减法
        if (signA == signB) {
            mantissaResult = mantissaA + mantissaB;
            if (mantissaResult >= (1 << 17)) { // 溢出，需要右移一位
                mantissaResult >>= 1;
                ++resultExponent;
            }
        } else {
            if (mantissaA > mantissaB) {
                mantissaResult = mantissaA - mantissaB;
                signB = signA;
            } else {
                mantissaResult = mantissaB - mantissaA;
                signA = signB;
            }
            // 左移直到尾数位为17位
            while ((mantissaResult & (1 << 16)) == 0 && mantissaResult != 0) { // 归一化
                mantissaResult <<= 1;
                --resultExponent;
            }
        }
        dev && cout << "resultExp " << resultExponent << endl;

        // 处理溢出, 设置为无穷
        if (resultExponent > 63) {
            result[0] = signA ? 0xFF : 0x7F;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 处理下溢
        if (resultExponent < -63) {
            result[0] = signA ? 0x80 : 0x00;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 编码结果
        result[0] = (signA ? 0x80 : 0x00) | ((resultExponent + 63) & 0x7F);
        result[1] = (mantissaResult >> 8) & 0xFF;
        result[2] = mantissaResult & 0xFF;
    }

    /**
     * 浮点数乘法
     * @a 乘数
     * @b 乘数
     * @result 结果
     */
    static void multiply(const uint8_t a[3], const uint8_t b[3], uint8_t result[3]) {
        // 提取符号位
        bool signA = a[0] & 0x80;
        bool signB = b[0] & 0x80;

        // 提取指数
        int exponentA = (a[0] & 0x7F) - 63;
        int exponentB = (b[0] & 0x7F) - 63;

        // 提取尾数 为 1.xx * 2^16
        uint32_t mantissaA = (1 << 16) | ((a[1] << 8) | a[2]);
        uint32_t mantissaB = (1 << 16) | ((b[1] << 8) | b[2]);

        // 计算结果的指数和尾数
        int resultExponent = exponentA + exponentB;
        uint64_t mantissaResult = static_cast<uint64_t>(mantissaA) * mantissaB;

        // 归一化处理
        // 由于尾数相乘是 (1.xx * 2^16) * (1.yy * 2^16)， 所以结果最多为1.zz * 2^33
        // 33位的结果，所以需要右移17位； 32位的结果，右移16位
        if (mantissaResult & (1ULL << (32 + 1))) { // 如果最高位为1，表示结果是33位
            mantissaResult >>= 1;
            ++resultExponent;
        }

        // 去掉额外的位数，保留规定的尾数位数
        mantissaResult >>= 16;

        // 处理溢出, 设置为无穷
        if (resultExponent > 63) {
            result[0] = signA ^ signB ? 0x80 : 0xFF;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 处理下溢
        if (resultExponent < -63) {
            result[0] = signA ^ signB ? 0x80 : 0x00;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 编码结果
        result[0] = (signA ^ signB ? 0x80 : 0x00) | ((resultExponent + 63) & 0x7F);
        result[1] = (mantissaResult >> 8) & 0xFF;
        result[2] = mantissaResult & 0xFF;
    }

    /**
     * 浮点数除法
     * @a 被除数
     * @b 除数
     * @result 结果
     */
    static void divide(const uint8_t a[3], const uint8_t b[3], uint8_t result[3]) {
        // 提取符号位
        bool signA = a[0] & 0x80;
        bool signB = b[0] & 0x80;

        // 提取指数
        int exponentA = (a[0] & 0x7F) - 63;
        int exponentB = (b[0] & 0x7F) - 63;

        // 提取尾数并加入隐含的前导1
        uint32_t mantissaA = (1 << 16) | ((a[1] << 8) | a[2]);
        uint32_t mantissaB = (1 << 16) | ((b[1] << 8) | b[2]);

        // 计算结果的符号位
        bool resultSign = signA ^ signB;

        // 计算结果的指数
        int resultExponent = exponentA - exponentB;
        dev && cout << endl << "规格化前exp " << resultExponent << endl;
        // 计算结果的尾数
        // 1.xx * 2^(16 + 17) / 1.yy * 2^16 = 1.zz * 2^17
        // 除数左移17位是防止结果有效位数小于17位
        uint64_t mantissaResult = ((static_cast<uint64_t>(mantissaA)) << 17) / mantissaB;
        resultExponent -= 1;
        dev && cout << "mantissaResult " << mantissaResult << endl;
        // 将尾数右移，直到为17位
        while (mantissaResult && mantissaResult >= (1ULL << 17)) {
            resultExponent ++;
            mantissaResult >>= 1;
        }
        dev && cout << endl << "resultExp " << resultExponent << endl;
        // 处理溢出, 设置为无穷
        if (resultExponent > 63) {
            result[0] = resultSign ? 0xFF : 0x7F;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 处理下溢
        if (resultExponent < -63) {
            result[0] = resultSign ? 0x80 : 0x00;
            result[1] = 0x00;
            result[2] = 0x00;
            return;
        }

        // 编码结果
        result[0] = (resultSign ? 0x80 : 0x00) | ((resultExponent + 63) & 0x7F);
        result[1] = (mantissaResult >> 8) & 0xFF; // 取高8位
        result[2] = mantissaResult & 0xFF;        // 取次高8位
    }

    // 将float24转换为十进制字符串
    std::string toString() {
        // 提取符号位
        bool sign = bytes[0] & 0x80;

        if (this->isNan()) {
            return "NaN";
        }

        if (this->isInf()) {
            return sign ? "-Inf" : "+Inf";
        }

        // 提取指数
        int exponent =(bytes[0] & 0x7F) - bias;

        // 提取尾数
        unsigned int mantissa = (bytes[1] << 8) | bytes[2];

        float fractionalMantissa = 1.0f;
        for (int i = 0; i < 16; ++i) {
            if (mantissa & (1 << (15 - i))) {
                fractionalMantissa += std::pow(2, -(i + 1));
            }
        }

        // 计算最终的十进制值
        float result = std::pow(2, exponent) * fractionalMantissa;
        if (sign) {
            result = -result;
        }

        return std::to_string(result);
    }

    friend std::ostream &operator<<(std::ostream &output, Float24 &f) {
        output << f.toString();
        return output;
    }

    Float24 operator+(Float24 & b) {
        if (this->isNan() || b.isNan()) {
            uint8_t result[3] = {0xFF, 0xFF, 0xFF};
            return Float24(result);
        }
        if (this->isInf() && b.isInf()) {
            uint8_t result[3] = {0xFF, 0x00, 0x00};
            return Float24(result);
        }
        if (this->isZero()) return Float24(b.getBytes());
        if (b.isZero()) return Float24(this->getBytes());
        uint8_t result[3] = {0, 0, 0};
        Float24::add(bytes, b.getBytes(), result);
        return Float24(result);
    }

    Float24 operator-(Float24 & b) {
        if (this->isNan() || b.isNan()) {
            uint8_t result[3] = {0xFF, 0xFF, 0xFF};
            return Float24(result);
        }
        if (this->isInf() && b.isInf()) {
            uint8_t result[3] = {0xFF, 0x00, 0x00};
            return Float24(result);
        }
        if (b.isZero()) return Float24(this->getBytes());
        uint8_t result[3] = {0, 0, 0};
        // 不改变b的情况下将b第一个字节第一位取反
        uint8_t* bBytes = b.getBytes();
        uint8_t bCopy[3] = {static_cast<uint8_t>(bBytes[0] ^ 0x80), bBytes[1], bBytes[2]};
        if (this->isZero()) return Float24(bCopy);
        Float24::add(bytes, bCopy, result);
        return Float24(result);
    }

    Float24 operator*(Float24 & b) const {
        uint8_t result[3] = {0, 0, 0};
        Float24::multiply(bytes, b.getBytes(), result);
        return Float24(result);
    }

    Float24 operator/(Float24 & b) const {
        uint8_t result[3] = {0, 0, 0};
        Float24::divide(bytes, b.getBytes(), result);
        return Float24(result);
    }

    Float24& operator=(const std::string& str) {
        *this = Float24(str);
        return *this;
    }

    Float24& operator=(const char* str) {
        return *this = std::string(str);
    }
};

int main() {
    string s1 = "3.14";
    string s2 = "0";
    Float24 f1 = s1;
    Float24 f2 = s2;
    cout << s1 + "转换为float24: \t", f1.printBytes();
    cout << s2 + "转换为float24: \t", f2.printBytes();
    Float24 f3;
    
    cout << (s1 + " + " + s2 + " = ") << (f3 = f1 + f2) <<  endl;
    cout << (s1 + " - " + s2 + " = ") << (f3 = f1 - f2) <<  endl;
    cout << (s1 + " * " + s2 + " = ") << (f3 = f1 * f2) <<  endl;
    cout << (s1 + " / " + s2 + " = ") << (f3 = f1 / f2) <<  endl;
    return 0;
}
```



## 参考

[IEEE 754 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/IEEE_754)

[**IEEE Standard for Floating-Point Arithmetic](https://trudbot-md-img.oss-cn-shanghai.aliyuncs.com/202406262130498.pdf)


