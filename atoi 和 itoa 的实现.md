# itoa 的实现

## atoi

```
int atoi(const char* str);
```

把一个字符串转换为 int 类型整数

```
int atoi(const char* str)
{
    const char* p = str;
    while (isspace(*p)) {
        ++p;
    }
    int sign = 1;
    if (*p == '-' || *p == '+') { // 处理正负号
        if (*p++ == '-') {
            sign = -1;
        }
    }

    if (!isdigit(*p)) {
        return 0;
    }

    long long i = 0;
    while (isdigit(*p)) {
        i = i * 10 + (*p++ - '0');
        if (sign * i >= INT_MAX) { // 处理溢出
            return INT_MAX;
        } else if (sign * i <= INT_MIN) {
            return INT_MIN;
        }
    }

    return sign * i;
}

int main()
{
    const char* p1 = "2147483647";
    printf("%d\n", atoi(p1));
    const char* p2 = "-2147483648";
    printf("%d\n", atoi(p2));
    const char* p3 = "0";
    printf("%d\n", atoi(p3)); 
    const char* p4 = "-0";
    printf("%d\n", atoi(p4));
    const char* p5 = "+1";
    printf("%d\n", atoi(p5));
}
```

输出：

```
2147483647
-2147483648
0
0
1
```

## itoa

```
char* itoa(int value, char* str, int base);
```

将 int 类型整数转换为进制 base 表示的字符串。

```
char* itoa(int value, char* str, int base)
{
    if (str == nullptr) {
        return nullptr;
    }

    char digits[32] = {
        'F', 'E', 'D', 'C', 'B', 'A',
        '9', '8', '7', '6', '5', '4', '3', '2', '1',
        '0',
        '1', '2', '3', '4', '5', '6', '7', '8', '9',
        'A', 'B', 'C', 'D', 'E', 'F'
    };

    const char* zero = digits + 15;
    int val = value;
    char* p = str;

    do {
        int rem = val % base;
        val /= base;
        *p++ = zero[rem];
    } while (val != 0);

    if (value < 0) {
        *p++ = '-';
    }

    std::reverse(str, p);
    *p = '\0';

    return str;
}

int main()
{
    char value[100];
    printf("%s\n", itoa(10, NULL, 10));
    printf("%s\n", itoa(0, value, 10));
    printf("%s\n", itoa(2147483647, value, 2));
    printf("%s\n", itoa(-2147483648, value, 2));
    printf("%s\n", itoa(-2560, value, 16));
    printf("%s\n", itoa(15, value, 5));
}
```

输出：

```
(null)
0
1111111111111111111111111111111
-10000000000000000000000000000000
-A00
30
```