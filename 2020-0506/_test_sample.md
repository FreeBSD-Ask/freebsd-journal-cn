# 测试文件

正文中有弯引号"这些"应保留。

行间代码中有弯引号 `echo "hello"` 应转换，注释中有弯引号 `# 这是 "注释"` 应保留。

```sh
# 这是"注释"，弯引号应保留
echo "hello"  # 行内注释 "注释" 也会被转换
VAR="world"
```

```c
/* 这是 "块注释" 的开始
   继续"注释"内容 */
int main() {
    const char *s = "string with "quotes"";
    return 0;
}
// 行注释 "注释" 应保留
```

```python
# 这是 "注释"，弯引号应保留
print("hello "world"")
```
