---
title: "設定の構成"
---

たくさんの _ScriptableObject_ から設定アセットを作成しますが、それらをどのような構成で管理するかについて整理しました。

## C#スクリプト内の構成

![イメージ](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/infrastructure-settings-csharp.png)

Infrastructure層に名前空間を定義し、それぞれ以下のようにSOを作成していきます。

- Settings
  -
- Settings.Domain
- Settings.Application
- Settings.Presentation
- Settings.Infrastructure

### 詳細

#### ScriptableSettingsクラス

すべての設定クラスが `UnityEngine.ScriptableObject` を直接継承するのではなく、ひとつ抽象クラスを挟むようにしています。

```csharp
namespace WordPuzzle.Infrastructure.Settings
{
    /// <summary>
    /// ScriptableObject設定基底クラス
    /// </summary>
    public abstract class ScriptableSettings : UnityEngine.ScriptableObject
    {
        ...
    }
}

namespace WordPuzzle.Infrastructure.Settings.Infrastructure.Report
{
    [CreateAssetMenu(
        fileName = nameof(SampleSettings),
        menuName = ... + nameof(ValidationReportSettings),
    )]
    public class SomeSettings : ScriptableSettings, ISomeSettings
    {
        ...
    }
}
```

私の場合、[Odin Inspector](https://odininspector.com/) を使うことが多いので、以下のように継承を差し替えることができます。

```csharp
namespace WordPuzzle.Infrastructure.Settings
{
    /// <summary>
    /// ScriptableObject設定基底クラス
    /// </summary>
    /// <remarks>
    /// <br/>- Odin Inspector の SerializedScriptableObject を継承しています。
    /// </remarks>
    public abstract class ScriptableSettings : Sirenix.OdinInspector.SerializedScriptableObject
    {
        ...
    }
}
```

## Unityプロジェクト内の配置

![イメージ](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/infrastructure-settings-unity.png)

```

```
