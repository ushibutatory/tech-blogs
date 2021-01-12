---
title: "NuGetパッケージをGitHubPackagesにPushしようとして404が返ってくる場合の対処"
emoji: "🔖"
type: "tech"
topics: ["GitHub","csharp","nuget","GitHubActions","GitHubPackages"]
published: true
---
# 実現したいこと

- C#でNuGetパッケージを作成する。
- GitHubでコードを管理する。
- GitHubActionsで、当該リポジトリのGitHubPackagesに配置する。

# 起きた事象

GitHubActions内にてNuGetパッケージをPushする。

- 環境変数
    - `${GITHUB_TOKEN}` = `${{ secrets.GITHUB_TOKEN }}`
    - `${NUGET_SOURCE}` = GitHubPackageのURL
      - 例: `"https://nuget.pkg.github.com/{organization-name}/index.json"`

```sh
dotnet nuget push '*.nupkg' -k ${GITHUB_TOKEN} -s ${NUGET_SOURCE} --skip-duplicate
```

404エラーが返ってくる。

```sh
Pushing {package-name}.nupkg to 'https://nuget.pkg.github.com/{organization-name}'...
  PUT https://nuget.pkg.github.com/{organization-name}/
  NotFound https://nuget.pkg.github.com/{organization-name}/ 377ms
error: Response status code does not indicate success: 404 (Not Found).
```

# 原因・対処

NuGetパッケージとなるC#プロジェクトの `.csproj` ファイルを確認します。

- RepositoryUrl
    - このURLがGitHubのリポジトリURLと一致していない場合、404エラーが返ってきます。

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.1</TargetFramework>
    ...
    <RepositoryUrl>https://github.com/{organization-name}/{repository-name}</RepositoryUrl>
    ...
  </PropertyGroup>

  <ItemGroup>
    ...
  </ItemGroup>
</Project>
```

# 備考

個人リポジトリでもOrganizationsリポジトリでも同様と思われます。
また、今回はGitHubActionsでのエラーでしたが、ローカルからCLIでPushしようとしても同様にエラーになります。

NuGetパッケージを作成する場合は、PropertyGroupをきちんと記述したほうがいいですね。

- [.NET Core の csproj 形式に追加されたもの # NuGet メタデータ プロパティ](https://docs.microsoft.com/ja-jp/dotnet/core/tools/csproj#nuget-metadata-properties)
