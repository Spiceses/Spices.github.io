+++

date = '2025-03-04T11:51:53+08:00'
draft = false
title = 'A Secure File Sharing System'
description = "加密文件共享系统"
image = "banner.jpg"
categories = ["cryptography", "file sharing", "security", "go"]

+++



这是一个有关如何在不安全的Cloud Database上与他人共享文件的项目, 在项目中我们将利用密码学知识设计合理的文件加密方案, 最终使得即使云数据库不安全, 我们也能安全地存储并与他人分享文件!

 ## Key technology

### Cryptography

* Cryptographic Hashes

* Symmetric-Key Encryption

* Message Authentication Codes (MACs)

## Design Overview

具体而言,  我们需要实现8个API

- `InitUser`: Given a new username and password, create a new user.
- `GetUser`: Given a username and password, let the user log in if the password is correct.
- `User.StoreFile`: For a logged-in user, given a filename and file contents, create a new file or overwrite an existing file.
- `User.LoadFile`: For a logged-in user, given a filename, fetch the corresponding file contents.
- `User.AppendToFile`: For a logged-in user, given a filename and additional file contents, append the additional file contents at the end of the existing file contents, while following some efficiency requirements.
- `User.CreateInvitation`: For a logged-in user, given a filename and target user, generate an invitation UUID that the target user can use to gain access to the file.
- `User.AcceptInvitation`: For a logged-in user, given an invitation UUID, obtain access to a file shared by a different user. Allow the recipient user to access the file using a (possibly different) filename of their own choosing.
- `User.RevokeAccess`: For a logged-in user, given a filename and target user, revoke the target user’s access so that they are no longer able to access a shared file.

### Users And User Authentication

Username x pwd -> User

第一部分是用户认证, 我们要实现`InitUser`与`GetUser`这两个API, 即密码学要回答的经典问题, 你如何向计算机证明你是"你", 答案也很自然, 就是使用只有用户知道的密码, 但是由于我们的密码是存储在不安全的云DB中的, 所以肯定需要加密, 这里采用了对称加密, User结构如下

```go
type User struct {
	Username string

    // Enc("pwdKey", pwd)
	Epwd []byte
}
```

对称加密的私钥暂且硬编码在代码中, 但是这仅仅保证了用户密码的Confidentiality, 如果有用户篡改了我的密码怎么办呢, 所以还需要用到MAC

```go
type User struct {
	Username string

    // Enc("pwdKey", pwd)
	Epwd []byte
    
    // HMAC(Username + Epwd)
	HMAC []byte
}
```

我选择简单地根据Username哈希值的前16位生成UUID, 并存储到DB的该UUID位置

#### Threat Model

现在想象一下Datastore Adversary可以发动什么攻击, 根据Kerckhoffs's Principle, 默认Datastore Adversary知道我们的源代码

* 如果尝试更改密码, 就需要知道Enc和MAC的私钥, 胡乱填充密码, 会导致登录失败
* 不知道私钥的情况, 只能使用拷贝攻击, 比如Alice注册了一个User "alice", Adversary注册了一个User "adversary", 那么Adversary可以把自己的Epwd字段替换掉Alice的Epwd, 由于我们Enc密码的私钥都是一样的, 这样做就避开了私钥这一难题, 我们是不是就可以用Adversary的密码登录Alice的账户了呢? 当然不是, 还有HMAC需要解决, 但此时又想到, 把HMAC也一起复制过去不就行了嘛, 遗憾的是, 还是不行, HMAC锁定了整个User, 它验证的是User的所有字段, 包括Username, 也就是说我们粘贴过去的HAMC是HMAC("adversary" + Epwd_adversary), 而Alice登陆时验证的是HMAC("alice" + Epwd_adversary), 看来这种攻击方法也失败了

### File Operations

这一部分是文件操作, 要实现的是`User.StoreFile`, `User.LoadFile`和`User.AppendToFile`, 最初的文件结构设计如下

```go
type File struct {
    // enc file content
    EncContent []byte
}
```

但是直接这样无法检测到文件的纂改, 所以我们同样要加上MAC

```go
type File struct {
    // enc file content
    EncContent []byte
    
    // MAC
    HMAC []byte
}
```

与此同时为了防止上文那样的拷贝攻击, 我们还需要其他的metadata, 由于这里的映射关系是
$$
User \times filename \rightarrow file content
$$
所以我们还需要存储username和filename

```go
type File struct {
	// file owner
	Owner string

	// H(fileName)
	HashFileName []byte
    
    // enc file content
    EncContent []byte
    
    // MAC
    HMAC []byte
}
```

文件存储的位置我选择随机生成一个UUID, 因此需要在userdata中加一个map来存储文件存储位置

```go
type User struct {
	Username string

    // Enc("pwdKey", pwd)
	Epwd []byte
    
    // HMAC(Username + Epwd)
	HMAC []byte
        
    // H(filename) -> file position
	FileUUID map[string]userlib.UUID
}
```

至此, 我大概明白了, 想要防御拷贝攻击, 需要存储的metadata应该从映射关系中得出.

由于`User.AppendToFile`要求时间复杂度是O(fileLength), 所以我选择以链表的形式存储文件, 那么文件就需要分页

```go
type File struct {
	// file owner
	Owner string

	// H(fileName)
	HashFileName []byte
    
    // enc file content
    EncContent []byte
    
    // current page
	Page int
    
    // MAC
    HMAC []byte
}
```

那么User.FileUUID就变成了一个二级指针, 它指向的是file的UUID列表

#### threat model

Enc和MAC保证了基本的防御, 那么剩下的我就只能想到拷贝攻击能起效了

* 攻击者把alice的文件aliceFile的内容替换为bob的文件bobFile, 我们的File.Owner字段可以检测出文件被替换
* 攻击者把alice的文件aliceFilePage1替换为aliceFilePage2, 我们可以检测File.Page来检测出页号出错

### Sharing and Revocation

终于到了重头戏, 文件的共享环节, 要实现的是`User.CreateInvitation`和`User.RevokeAccess`, 就目前的结构来看, 所有知道私钥的人都可以解密文件, 不过还有文件拥有者和文件名的验证, 使得只有拥有者的账号才能访问文件, 所以我们创建的邀请中除了文件存储位置还要包含文件拥有者和文件名

```go
type Invitation struct {
	Owner string
    
    OwnerHashFilename []byte

	FileUUID userlib.UUID

	HMAC []byte
}
```

那么撤销操作呢, 我们如何保证拥有者撤销分享之后接收者不能再获取文件, 这是一个比听上去稍微复杂一些的问题. 考虑以下情况

![proj2-spec-share-diagram](https://assets.cs161.org/proj2/proj2-spec-share-diagram.png)

A -> B指A把文件分享给了B, 如上图所示, 如果A撤销了对B的文件分享, 我们如何保证BDEF不能再读取文件, 如何保证CG还能读取文件呢

实际上这里我有两个想法, 一个是更改文件的存储位置, 另一个是对文件进行二次加密, 然后在分享时把密钥分发给接收者, 撤销时更改密钥重新加密文件, 这里我选择了第二种做法, 因此邀请函结构如下

```go
type Invitation struct {
	Owner string

	OwnerHashFilename []byte

	FileUUID userlib.UUID

	ShareKeyUUID userlib.UUID

	HMAC []byte
}
```

值得一提的是, 我们这里的shareKey不是直接存储在邀请函中的, 而是另外分配了一个UUID存储在Datastore中的, 我们只为每个文件开辟一块shareKey的位置, 也就是说上面的图中的所有收到文件的用户BCDEFG用的shareKey都存储在一个UUID上, 这样的话假如我要撤销对B的分享, 那么我需要做的只是换一个UUID存储一个新的shareKey, 然后更改发送给C的邀请函, 而不需要再去修改后续的发送给G的邀请函了, 因为此时C, G共享一个shareKey

还有一个细节是接收者收到文件后会给文件一个别名, 之后load file时会使用别名来获取文件, 所以我们再在邀请函中加上file alias

```go
type Invitation struct {
	Owner string

	OwnerHashFilename []byte
    
    HashFileAlias []byte

	FileUUID userlib.UUID

	ShareKeyUUID userlib.UUID

	HMAC []byte
}
```

这一部分我后知后觉地意识到, 原来私钥不止可以用来防御攻击者, 实际上可以用来防御任何人, 比如这里, 我们有一个所有用户共享的私钥来防御DB攻击者, 还有一个每个用户文件持有一个的私钥来防御未收到分享的人

#### threat model

* 如果alice把file1分享给bob, 把file2分享给jim, 攻击者充满恶意地把两个uuid的内容更换了一下, 按照上面的方案bob就在不知不觉中获得了file2, jim获得了file1, 因此我们再加一个receiver字段
  ```go
  type Invitation struct {
  	Owner string
  
  	OwnerHashFilename []byte
      
      HashFileAlias []byte
  
  	FileUUID userlib.UUID
  
  	ShareKeyUUID userlib.UUID
      
      Receiver string
  
  	HMAC []byte
  }
  ```

## Sumarry

把上述设计思路付诸实践之后, 我得到了一个大约1500行代码的项目, 我们的系统基本可以做到在不安全的数据库上分享文件了, 具体来说, 它可以防御如下攻击:

* 攻击者直接纂改文件内容
* 攻击者用一个用户的文件替换另一个用户的文件
* 被撤销共享权限的用户尝试重新获得文件读取修改权限

总的来说, 这个项目是一次很好的密码学应用实践, 在这个项目的实现过程中, 我深感自身软件工程能力的不足, 比如:

* 不知道如何系统地组织测试
* 不知道如何设计好的数据结构

但是也收获了许多, 我学到了:

* 密码学设计基础
* go的基本语法, Ginko测试框架的使用

