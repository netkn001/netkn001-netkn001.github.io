---
title: rust newtype 实践
date: 2024-11-07
tags:
  - rust
---

`newtype` 就是使用元组结构体的方式将已有的类型包裹起来， 形如 `struct Wrapper(T)`。

那 `newtype` 有什么用呢？

孤儿规则：只有在 `crate` 中定义了 `trait` 或类型时，才能为类型实现 `trait`。

因这孤儿规则的限制，因此想对引用库做些扩展是做不到的， 这就需要 `newtype` 来拯救了，配合如 `Defer` 就可以很好扩展。

如
```rust
struct Wrapper(Vec<String>);

impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}

impl Deref for Wrapper {
    type Target = Vec<String>;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let v = Wrapper(vec!["hello".to_string(), "world".to_string()]);
    println!("{v}");
}
```
就可以对 `Vec` 做 `to_string()` 的扩展。

那 `newtype` 只有这个妙用吗？

当然不是，`rust` 作为一个类型安全的一门语言，`newtype` 在类型安全上也是大放异彩。

## newtype 驱动设计

看如下非常常见的示例

```rust
pub fn create_user(email: &str, password: &str) -> Result<User, CreateUserError> {
    validate_email(email)?;
    validate_password(password)?;
    let password_hash = hash_password(password)?;
    // ...
    Ok(User)
}
```
很熟悉是不是，或多或少都写过类似的函数。

先看 `input` 即入参，接收两个参数 `email` 和 `password`，调用的时候弄错顺序是容易出现的，如果多几个接收相关类型的参数呢？ 对调用者来说绝对是个坑，如果再过段时间就更可怕了，绝对是一个定时炸弹，维护恶梦。

再看代码里面的 `validate`，需要验证不同的场景，返回不同的错误类型

```rust
#[derive(Error, Clone, Debug, PartialEq)]
pub enum CreateUserError {
    #[error("invalid email address: {0}")]
    InvalidEmail(String),
    #[error("invalid password: {reason}")]
    InvalidPassword {
        reason: String,
    },
    #[error("failed to hash password: {0}")]
    PasswordHashError(#[from] BcryptError),
    #[error("user with email address {email} already exists")]
    UserAlreadyExists {
        email: String,
    },
    // ...
}
```

然后单元测试做覆盖

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_create_user_invalid_email() {
        let email = "invalid-email";
        let password = "password";
        let result = create_user(email, password);
        let expected = Err(CreateUserError::InvalidEmail(email.to_string()));
        assert_eq!(result, expected);
    }

    #[test]
    fn test_create_user_invalid_password() { unimplemented!() }

    #[test]
    fn test_create_user_password_hash_error() { unimplemented!() }

    #[test]
    fn test_create_user_user_already_exists() { unimplemented!() }

    #[test]
    fn test_create_user_success() { unimplemented!() }
}
```
这看起来很合理，是很合理，但是，会让业务逻辑不聚集。

再回头看看单元测试，验证与业务的测试是在一起的，这真的是好事吗？不是好事，单元测试是可以反哺我们做更好的设计。

现在到 `newtype` 出场了。

增加定义

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct EmailAddress(String);

#[derive(Debug, Clone, PartialEq)]
pub struct Password(String);
```

强制类型了，规避顺序问题了，调用方也明朗了。

剩下的就是验证了，要怎么做？把如上验证的那部分抽出来，放到各自的类型中验证，也是单一职责的体现。

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct EmailAddress(String);

#[derive(Error, Debug, Clone, PartialEq)]
#[error("{0} is not a valid email address")]
pub struct EmailAddressError(String);

impl EmailAddress {
    pub fn new(raw_email: &str) -> Result<Self, EmailAddressError> {
        if email_regex().is_match(raw_email) {
            Ok(Self(raw_email.into()))
        } else {
            Err(EmailAddressError(raw_email.to_string()))
        }
    }
}

#[derive(Error, Clone, Debug, PartialEq)]
#[error("a user with email address {0} already exists")]
pub struct UserAlreadyExistsError(EmailAddress);

fn create_user(email: EmailAddress, password: Password) -> Result<User, UserAlreadyExistsError> {
    // ...10
    Ok(User)
}
```

对应单元测试也可以拆分了，关注点也分离了，测试也更聚集了。

```rust
#[cfg(test)]
mod email_address_tests {
    use super::*;

    #[test]
    fn test_new_invalid_email() {
        let raw_email = "invalid-email";
        let result = EmailAddress::new(raw_email);
        let expected = Err(EmailAddressError(raw_email.to_string()));
        assert_eq!(result, expected);
    }

    #[test]
    fn test_new_success() {
        unimplemented!()
    }
}

#[cfg(test)]
mod create_user_tests {
    use super::*;

    #[test]
    fn test_user_already_exists() {
        unimplemented!()
    }

    #[test]
    fn test_success() {
        unimplemented!()
    }
}
```

如上，做了一个转变就是将验证变更成解析。

> Parse, don't validate

验证和解析之前区别在于如何保存信息，两者的主要任务还是验证数据。

解析器是一个将结构化程度较低的输入，转换为结构化程序较高的输出，伴随解析失败的错误。

解析器是一个非常强大的工具：它们允许在程序和外部世界之间的边界上预先对输入进行检查，并且一旦执行了这些检查，就不再需要再次检查。

好好体会下。

## newtype 很重要的相关 trait

为新类型派生有意义的特性是一种好行为，即使没有使用到，特别是在开发库的时候（因为孤儿规则的限制，使用者无法使用到这个派生特性）

如果想让新类型的行为与基础类型类似，`#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]` 这几个可以按需要派生。

可以实现对应的 `From` 或者 `TryFrom` 可以很方便地对类型做转换

```rust
impl TryFrom<&str> for EmailAddress {
    type Error = EmailAddressError;

    fn try_from(value: &str) -> Result<Self, Self::Error> {
        EmailAddress::new(value)
    }
}
```

`AsRef`, `Deref`, `Borrow` 可以很方便地将新类型的使用体验变成原始类型的使用。

```rust
impl AsRef<str> for EmailAddress {
    fn as_ref(&self) -> &str {
        &self.0
    }
}

impl Deref for EmailAddress {
    type Target = str;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl Borrow<str> for EmailAddress {
    fn borrow(&self) -> &str {
        &self.0
    }
}
```

## 消除样板代码

如上示例，一个新类型其实是做了很多事，实际项目中，新类型会很多，这样手动实现这些代码其实是很麻烦的，好在现在有些库可以让我们减少这些样板工作。

- [derive_more](https://docs.rs/derive_more/latest/derive_more/)
- [nutype](https://docs.rs/nutype/latest/nutype/)
- [refined_type](https://github.com/tomoikey/refined_type)
