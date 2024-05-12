---
title: "Nuxt3 + .NET Coreで作るセッションログイン機能サンプル：1. .NET Coreの立ち上げ・ユーザーCRUD操作実装"
emoji: "🫠"
type: "tech"
topics: [dotnet, nuxt3]
published: true
---
Nuxt3をフロント、.NET Coreをバックエンドとしてセッションログイン機能を作成する作業備忘録。
# モチベーション
## Nuxt3をViewとしたときに.NET Coreでログイン機能を実装したい
.NET CoreのMVCの場合はEntity Frame CoreとIdentity.EntityFrameworkCoreがあればログイン機能を簡単に実装することができる。
一方、Nuxt3などViewにフレームワークを使うとログイン機能連携に少し工数が増えるため、実際のデプロイ環境を用いて構築する。

# .NET Coreの立ち上げ

## .NET Coreプロジェクト作成
.NET Core APIのプロジェクトを立ち上げる。

```console
dotnet new webapi -n <project-name>
```

## ログイン機能に必要なパッケージのインストール
セッションログインに必要なパッケージをインストールする。
NugetまたはCLIよりインストールする。
### EntityFrameworkCore, EntityFrameworkCoreTools
DB操作に必要なパッケージ。
```console
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

### AspNetCore.Identity.EntityFrameworkCore
ユーザー認証に関するパッケージ。
```console
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

### EntityFrameworkCore.SqlServer
SQL Serverと接続するパッケージ。
```console
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

# .NET CoreとDB接続
## ユーザー・ロールModelを登録
追加したパッケージのクラスIdentityUserとIdentityRoleを使用することにより
ユーザー認証や、ロール認証によるAPIコントローラの認可することができる。

ユーザーを登録するDBを作成するため、はじめにModelを設定する。
AspNetCore.Identity.EntityFrameworkCoreをインストールしたあと、dotnet efなどによりMigrationすることで、
設定したDBに新しくAspNetUserテーブルが作成されユーザー情報を格納することができる。
今回は、AspNetUserに追加のユーザー情報を登録するため、ApplicationUserとApplicationRoleとして継承する。

```diff
back(NET Coreプロジェクト)
+  |--Models
+  |     |--ApplicationRole.cs
+  |     |--ApplicationUser.cs
   |--bin
   |--obj
   |--Properties
   |--Program.cs
```

```csharp:ApplicationUser.cs
using Microsoft.AspNetCore.Identity;

namespace back.Models;

public class ApplicationUser: IdentityUser<Guid>
{
    // 追加情報例
    public string Job { get; set; }
}
```

```csharp:ApplicationRole.cs
using Microsoft.AspNetCore.Identity;

namespace back.Models;

public class ApplicationRole: IdentityRole<Guid>
{
    // 必要に応じてメンバ追加
}
```

## DbContextの設定
DbContextを追加し、先ほど作成したModelを設定する。
また、DbContextに関してもIdentityDbContextを継承する。

```diff
back(NET Coreプロジェクト)
+  |--Data
+  |     |--ApplicationDbContext.cs
   |--Models
   |     |--ApplicationRole.cs
   |     |--ApplicationUser.cs
   |--bin
   |--obj
   |--Properties
   |--Program.cs
```

```cs:ApplicationRole.cs
namespace back.Data;

public class ApplicationDbContext: IdentityDbContext<ApplicationUser, ApplicationRole, Guid>
{

    public ApplicationDbContext()
    {}

    public DbSet<ApplicationUser> ApplicationUsers { get; set; }
    public DbSet<ApplicationRole> ApplicationRoles { get; set;}

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
    }
}
```

## DBと接続する
Program.csにSQL Serverとの接続を定義する。
接続対象のDbContextはApplicationDbContextとし、接続情報をappsettings.jsonに記載する。

```diff json:appsetting.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
+  "ConnectionStrings": {
+    "DefaultConnection": "<接続情報>"
  },
}
```

```diff csharp:Program.cs
using back.Data;
using back.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;


// -----------------------------
// Builder Settings
// -----------------------------
var builder = WebApplication.CreateBuilder(args);

// API Controller
builder.Services.AddControllers();

// Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

+// DB Connection
+builder.Services.AddDbContext<ApplicationDbContext>(options =>
+    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));


// -----------------------------
// App Settings
// -----------------------------
var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
else
{
    app.UseHttpsRedirection();
}

// Authentication and Authorization
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```


## MigrationによるDBテーブル作成
SQL Serverに設定したModel（AspNet）のテーブルを追加する。
コマンドはEntityFrameworkCore.Toolsのdotnet efからmigrationする。
```console
dotnet ef add migration <Migration名> --output Data/Migrations
```
Data/Migrations内にDBを更新するC#のコードが作成されるため、
その内容をSQL Serverに反映させる。
```console
dotnet ef database update
```

databaseの更新後にSQL Serverにてテーブルが作成された確認する。

# ユーザーのCRUD
作成されたAspNetUserテーブルにて、ユーザーのCRUDを実際に行う。
ControllersディレクトリにUserManageController.csを作成し、各APIからユーザー操作を行う。

```cs:UserManageController.cs
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace back.Controller;

[ApiController]
[Route("[controller]")]
public class UserManageController: ControllerBase
{

    private readonly UserManager<ApplicationUser> _userManager;
    public UserManageController(
        UserManager<ApplicationUser> userManager,
    )
    {
        _userManager = userManager;
    }
}
```

また、各APIへのリクエストを新規にDTOディレクトリ内に作成する。
今回はユーザー作成、ユーザー削除、ユーザー編集のリクエストを作成する。

```diff
back(NET Coreプロジェクト)
+  |--Controller
+  |     |--UserManageController.cs
+  |--DTO
+  |     |--UserCreateRequest.cs
+  |     |--UserDeleteRequest.cs
+  |     |--UserEditRequest.cs
   |--Data
   |     |--ApplicationDbContext.cs
   |--Models
   |     |--ApplicationRole.cs
   |     |--ApplicationUser.cs
   |--bin
   |--obj
   |--Properties
   |--Program.cs
```


## Create
ユーザー作成用のリクエスト内容を設定する。
リクエスト内容はUserCreateRequest.csにメンバとして登録する。

```cs:UserCreate.cs
namespace back.DTO;

public class UserCreateRequest
{
    public string UserName {get; set;}
    public string Email {get; set;}
    public string Password {get; set;}
    // ApplicationUserに追加したメンバ
    public string Job {get; set;}
}
```

続いてUserManageController.csにユーザー作成用のAPIを追加する。
今回はバリデーションはすべてバックエンド側を想定した実装になっている。

```cs:UserManageController.cs
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace back.Controller;

[ApiController]
[Route("[controller]")]
public class UserManageController: ControllerBase
{

    private readonly UserManager<ApplicationUser> _userManager;
    public UserManageController(
        UserManager<ApplicationUser> userManager,
    )
    {
        _userManager = userManager;
    }

    [HttpPost("UserCreate")]
    public async Task<IActionResult> UserCreate(UserCreateRequest userCreateRequest)
    {
        // 既存の名前があるか
        var existingUser = await _userManager.FindByNameAsync(userCreateRequest.UserName);
        if (existingUser != null) return BadRequest(new { message = "すでに同じ名前が登録されています" });

        // 既存のメールアドレスがあるか
        var existingUserByEmail = await _userManager.FindByEmailAsync(userCreateRequest.Email);
        if (existingUserByEmail != null) return BadRequest(new { message = "すでに同じメールアドレスが登録されています" });

        // ApplicationUserインスタンス作成
        // 追加したプロパティをセット
        ApplicationUser user =new ApplicationUser
        {
          Job = userCreateRequest.Job
        }

        // 名前・メールアドレスの設定
        await _userStore.SetUserNameAsync(user, registerRequest.UserName, CancellationToken.None);
        await _userStore.SetUserEmailAsync(user, registerRequest.Email, CancellationToken.None);

        // ユーザー作成
        var result = await _userManager.CreateAsync(user, registerRequest.Password);
        if (!result.Succeeded) return BadRequest(new { message = "ユーザー作成に失敗しました" });

        return Ok(new { message = "ユーザー作成に成功しました" });
    }

}
```

## Read
UserManageController.csに登録したユーザーを取得するAPIを実装する。
ユーザーすべてまたは特定のIDを持つシングルユーザーをGETメソッドで取得する。

```cs:UserManageController.cs
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace back.Controller;

[ApiController]
[Route("[controller]")]
public class UserManageController: ControllerBase
{

    private readonly UserManager<ApplicationUser> _userManager;
    public UserManageController(
        UserManager<ApplicationUser> userManager,
    )
    {
        _userManager = userManager;
    }

    [HttpGet("SingleUser/{id}")]
    public async Task<IActionResult> SingleUser(string id)
    {
        var user = await _userManager.FindByIdAsync(id);
        if (user == null)　return NotFound(new { message = "IDを持つユーザーが見つかりませんでした" });

        return Ok(user);
    }

    [HttpGet("AllUser")]
    public async Task<IActionResult> AllUser()
    {
        var users = await _userManager.Users.ToListAsync();
        if (users == null)　return NotFound(new { message = "ユーザーが見つかりませんでした" });

        return Ok(users);
    }
}


## Update
ユーザー更新用のリクエスト内容を設定する。
リクエスト内容はUserUpdateRequest.csにメンバとして登録する。

```cs:UserUpdateRequest.cs
namespace back.DTO;

public class UserUpdateRequest
{
    public Guid UserId {get; set;}
    public string UserName {get; set;}
    public string Email {get; set;}
    public string Password {get; set;}
    // ApplicationUserに追加したメンバ
    public string Job {get; set;}
}
```

続いてUserManageController.csにユーザー更新用のAPIを追加する。
ユーザー作成同様にバリデーションはすべてバックエンド側を想定した実装になっている。
また、ユーザー情報の更新とパスワード更新はそれぞれ個別のトランザクションのため、
DbContextを用いてユーザー編集のトランザクションを一括に管理する。

```cs:UserManageController.cs
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace back.Controller;

[ApiController]
[Route("[controller]")]
public class UserManageController : ControllerBase
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ApplicationDbContext _dbContext;

    public UserManageController(UserManager<ApplicationUser> userManager, ApplicationDbContext dbContext)
    {
        _userManager = userManager;
        _dbContext = dbContext;
    }

    [HttpPatch("UserUpdate")]
    public async Task<IActionResult> UserUpdate(UserUpdateRequest userUpdateRequest)
    {
        using (var transaction = _dbContext.Database.BeginTransaction())
        {
            try
            {
                // 編集するユーザーIDを取得
                var user = await _userManager.FindByIdAsync(userUpdateRequest.Id);
                if (user == null) return BadRequest(new { message = "IDを持つユーザーが見つかりませんでした" });

                // 既存の名前と被らないか確認
                var existingUser = await _userManager.FindByNameAsync(userUpdateRequest.UserName);
                if (existingUser != null && existingUser.Id != user.Id) return BadRequest(new { message = "すでに同じ名前が登録されています" });

                // 既存のメールアドレスと被らないか確認
                existingUser = await _userManager.FindByEmailAsync(userUpdateRequest.Email);
                if (existingUser != null && existingUser.Id != user.Id) return BadRequest(new { message = "すでに同じメールアドレスが登録されています" });

                // ApplicationUser情報を更新
                user.UserName = userUpdateRequest.UserName;
                user.Email = userUpdateRequest.Email;
                user.Job = userUpdateRequest.Job;

                // ユーザー情報を更新
                var updateResult = await _userManager.UpdateAsync(user);
                if (!updateResult.Succeeded) return BadRequest(new { message = "ユーザー情報更新に失敗しました" });

                // パスワードを更新
                var token = await _userManager.GeneratePasswordResetTokenAsync(user);
                var passwordResult = await _userManager.ResetPasswordAsync(user, token, userUpdateRequest.Password);
                if (!passwordResult.Succeeded) return BadRequest(new { message = "ユーザーパスワード変更に失敗しました" });

                // トランザクションをコミット
                transaction.Commit();
                return Ok(new { message = "ユーザー更新に成功しました" });
            }
            catch (Exception ex)
            {
                transaction.Rollback();
                return BadRequest(new { message = $"ユーザー編集に失敗したためロールバックを実行しました: {ex.Message}" });
            }
        }
    }
}
```

## Delete
ユーザー削除用のリクエスト内容を設定する。
リクエスト内容はUserDeleteRequest.csにメンバとして登録する。

```cs:UserUpdateRequest.cs
namespace back.DTO;

public class UserUpdateRequest
{
    public Guid UserId {get; set;}
}
```

続いてUserManageController.csにユーザー削除用のAPIを追加する。

```cs:UserManageController.cs
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace back.Controller;

[ApiController]
[Route("[controller]")]
public class UserManageController: ControllerBase
{

    private readonly UserManager<ApplicationUser> _userManager;
    public UserManageController(
        UserManager<ApplicationUser> userManager,
    )
    {
        _userManager = userManager;
    }

    [HttpDelete("UserDelete/{id}")]
    public async Task<IActionResult> UserDelete(UserDeleteRequest userDeleteRequest)
    {
        var user = await _userManager.FindByIdAsync(userDeleteRequest.UserId);
        if (user == null) return NotFound();

        var result = await _userManager.DeleteAsync(user);

        return Ok(new { message = "ユーザー削除に成功しました" });
    }
}
```

# Nuxt 3との接続
次回は登録したユーザーからセッションログインを設定する。

