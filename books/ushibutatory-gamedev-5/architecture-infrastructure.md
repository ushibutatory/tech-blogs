---
title: "アーキテクチャ詳細 - Infrastructure層"
---

## Infrastructure層

- すること
  - 各レイヤーで実装を避けて抽象化したインタフェースの実装を定義する。
- しないこと
  - 他レイヤーの参照はするが、他レイヤーのコンポーネント等の操作はしない。

## 詳細

### Adapter

抽象化したインタフェースの実装です。

- 例）
  - 各種リポジトリインタフェース
  - 乱数インタフェース
  - 時計インタフェース

::: details 例

```csharp
namespace WordPuzzle.Infrastructure.Adapters.Domain.PlayerProfile
{
    /// <summary>
    /// プレイヤープロファイルリポジトリ：ローカルファイル向け
    /// </summary>
    /// <remarks>
    /// <br/>- 開発用
    /// </remarks>
    public class LocalPlayerProfileRepository : Repository<LocalPlayerProfileRepository>, IPlayerProfileRepository
    {
        [Inject] private readonly IPlayerProfileSettings _settings = default!;

        private string _FilePath => Path.Combine(UnityEngine.Application.persistentDataPath, _settings.LocalFileName);

        public Task<IReadOnlyList<WordPuzzle.Domain.PlayerProfile.PlayerProfile>> LoadAllAsync()
            => Task.FromResult(_LoadAll());

        private IReadOnlyList<WordPuzzle.Domain.PlayerProfile.PlayerProfile> _LoadAll()
        {
            var filePath = _FilePath;
            if (!File.Exists(filePath))
            {
                _logger?.LogInformation("ファイルが存在しないため空リストを返します。");
                return new List<WordPuzzle.Domain.PlayerProfile.PlayerProfile>();
            }

            _logger?.LogInformation("ファイルをロードします ... : {Path}", filePath);
            try
            {
                var json = File.ReadAllText(filePath, Encoding.UTF8);
                var bytes = Encoding.UTF8.GetBytes(json);
                var dto = SerializationUtility.DeserializeValue<PlayerProfileListDto>(bytes, DataFormat.JSON);

                _logger?.LogInformation("ファイルをロードしました。");
                return PlayerProfileListDto.ToDomain(dto);
            }
            catch (System.Exception ex)
            {
                _logger?.LogError(ex, "ファイルのロード中にエラーが発生しました。");
                return new List<WordPuzzle.Domain.PlayerProfile.PlayerProfile>();
            }
        }

        public async Task SaveAsync(WordPuzzle.Domain.PlayerProfile.PlayerProfile profile)
        {
            var profiles = _LoadAll().ToList();
            var index = profiles.FindIndex(p => p.ProfileId == profile.ProfileId);
            if (index >= 0)
                profiles[index] = profile;
            else
                profiles.Add(profile);

            await _SaveAsync(_FilePath, profiles);
        }

        public async Task DeleteAsync(PlayerProfileId id)
        {
            var profiles = _LoadAll().Where(p => p.ProfileId != id).ToList();

            await _SaveAsync(_FilePath, profiles);
        }

        private async UniTask _SaveAsync(string filePath, List<WordPuzzle.Domain.PlayerProfile.PlayerProfile> profiles)
        {
            _logger?.LogInformation("ファイルを保存します ... : {Path}", filePath);

            var dto = PlayerProfileListDto.ToDto(profiles);
            var bytes = SerializationUtility.SerializeValue(dto, DataFormat.JSON);
            var json = Encoding.UTF8.GetString(bytes);

            await File.WriteAllTextAsync(filePath, json, Encoding.UTF8);

            _logger?.LogInformation("ファイルを保存しました。");
        }
    }
}
```

:::

### InfrastrucureSettingsインタフェース

インフラ層の挙動を指定する設定群です。

- 例）
  - 外部アクセスのAPIキー、等

他レイヤーのSettingsインタフェースと考え方は同じです。

### Settingsクラス

各レイヤーで抽象化したインタフェースの実装のうち、ScriptableObjectで定義するものです。

::: details 例

```csharp
namespace WordPuzzle.Infrastructure.Settings.Domain.Score
{
    [CreateAssetMenu(
        fileName = nameof(BasicScoreSettings),
        menuName = MENU + "Domain/Score/" + nameof(BasicScoreSettings),
        order = (int)Order.Domain__Score
    )]
    public class BasicScoreSettings : ScoreSettings, IBasicScoreSettings
    {
        public override ScoreType ScoreType => ScoreType.Basic;

        [SerializeField] private bool _enabled = default!;
        public bool Enabled => _enabled;

        [SerializeField] private int _point = default!;
        public ScorePoint ScorePoint => ScorePoint.Of(_point);
    }
}
```

:::

[別ページ](infrastructure-settings)に分割しました。

## 所感

以前はInfrastructure層はプレゼンテーション層を参照していませんでしたが、今回は参照するようにしました。プレゼンテーション層の挙動制御用のSOを定義する場所が欲しかったからです。結果的にこうしてよかったと思います。
