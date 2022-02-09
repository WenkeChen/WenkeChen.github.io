---
title: "关于CSRF与表单重复提交"
date: 2022-02-09T15:50:59+08:00
draft: false
---

> 跨站请求伪造（英语：Cross-site request forgery），也被称为 one-click attack 或者 session riding，通常缩写为 CSRF 或者 XSRF， 是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法。跟跨网站脚本（XSS）相比，XSS 利用的是用户对指定网站的信任，CSRF 利用的是网站对用户网页浏览器的信任。

### 问题

抵御跨站请求伪造和表单重复提交最有效的手段都是在表单中加一个token，那么CSRF的token能用于防御表单重复提交吗？

### 求证

扒了一下symfony关于CSRF token的生成源码。

通过tokenId获取token
```php
public function getCsrfToken(string $tokenId): string
{
    return $this->csrfTokenManager->getToken($tokenId)->getValue();
}
```

如果存储中存在对应的token就直接拿，否则创建
```php
public function getToken($tokenId)
{
    $namespacedId = $this->getNamespace().$tokenId;
    if ($this->storage->hasToken($namespacedId)) {
        $value = $this->storage->getToken($namespacedId);
    } else {
        $value = $this->generator->generateToken();

        $this->storage->setToken($namespacedId, $value);
    }

    return new CsrfToken($tokenId, $value);
}
```

真正的生成的方法
```php
class UriSafeTokenGenerator implements TokenGeneratorInterface
{
    private $entropy;

    /**
     * Generates URI-safe CSRF tokens.
     *
     * @param int $entropy The amount of entropy collected for each token (in bits)
     */
    public function __construct(int $entropy = 256)
    {
        $this->entropy = $entropy;
    }

    /**
     * {@inheritdoc}
     */
    public function generateToken()
    {
        // Generate an URI safe base64 encoded string that does not contain "+",
        // "/" or "=" which need to be URL encoded and make URLs unnecessarily
        // longer.
        $bytes = random_bytes($this->entropy / 8);

        return rtrim(strtr(base64_encode($bytes), '+/', '-_'), '=');
    }
}
```
可以看出，token实际上是32位定长（默认情况）字符经过base64编码后的字符串（替换了字符及删除末尾的“=”），这里并不能看出token能用于防止表单重复提交（是不是每次用完之后即失效，或者每次都会重新生成tokenId）。

再看表单生成tokenId的地方：
```php
/**
 * Adds a CSRF field to the form when the CSRF protection is enabled.
 */
public function buildForm(FormBuilderInterface $builder, array $options)
{
    if (!$options['csrf_protection']) {
        return;
    }

    $builder
        ->addEventSubscriber(new CsrfValidationListener(
            $options['csrf_field_name'],
            $options['csrf_token_manager'],
            $options['csrf_token_id'] ?: ($builder->getName() ?: \get_class($builder->getType()->getInnerType())),
            $options['csrf_message'],
            $this->translator,
            $this->translationDomain,
            $this->serverParams
        ))
    ;
}
```

从这里可以看出，csrf_token_id用的是表单的名字或者表单类名，无论是哪一种，id都是固定的。

## 总结

在一次存储（session|redis）的生命周期内，每次加载form其CSRF token都是固定的，CSRF token不能用于防止表单重复提交。要想CSRF token用于防止表单重复提交那就得每次表单提交后删除其对应的token了；

当然以上针对的只是标准搞法，不保证所有地方都这样。


