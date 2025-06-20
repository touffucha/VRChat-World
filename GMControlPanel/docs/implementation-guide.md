# GMコントロールパネル 実装手順書

## 概要
本手順書は、[要件定義書](./requirements.md)に基づいてTRPG用GMコントロールパネルを実装するための段階的なガイドです。

## 前提条件
- Unity 2022.3.6f1（VRChat推奨バージョン）
- VRChat SDK3-Worlds
- UdonSharp 1.1.x以上
- TextMeshPro（UI表示用）

## 実装手順

### フェーズ1: 基本構造の構築

#### 1.1 プロジェクト構造の準備
```
GMControlPanel/
├── Prefabs/
│   ├── GMControlPanel.prefab      # メインパネル
│   ├── WristPanel.prefab          # 手首パネル
│   └── Components/                # UI部品
├── Scripts/
│   ├── Core/                      # コアシステム
│   ├── UI/                        # UI制御
│   └── Gimmicks/                  # ギミック制御
├── Materials/                     # UI用マテリアル
└── Textures/                      # アイコン・背景
```

#### 1.2 基本Canvasの設定
1. **手首パネル用Canvas作成**
   - Hierarchy右クリック → UI → Canvas
   - Canvas名を`WristCanvas`に変更
   - Canvas Component設定:
     - Render Mode: `World Space`
     - サイズ: 80×60（要件3.2準拠）
   - VRC_UiShapeコンポーネントを追加

2. **メインパネル用Canvas作成**
   - 同様の手順で`MainCanvas`を作成
   - サイズ: 400×300（要件3.2準拠）

### フェーズ2: コアシステムの実装

#### 2.1 GM認証システム
`Scripts/Core/GMAuthenticator.cs`を作成:

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

public class GMAuthenticator : UdonSharpBehaviour
{
    [SerializeField] private string[] gmPlayerNames;
    [SerializeField] private GameObject controlPanelRoot;
    
    void Start()
    {
        // GM判定とパネル表示制御
        if (IsGM(Networking.LocalPlayer))
        {
            controlPanelRoot.SetActive(true);
        }
    }
    
    private bool IsGM(VRCPlayerApi player)
    {
        // GM判定ロジック
        foreach (string gmName in gmPlayerNames)
        {
            if (player.displayName == gmName)
                return true;
        }
        return false;
    }
}
```

#### 2.2 パネル追従システム
`Scripts/Core/PanelFollower.cs`を作成:

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

public class PanelFollower : UdonSharpBehaviour
{
    [SerializeField] private Transform targetPanel;
    [SerializeField] private float followDistance = 0.5f;
    [SerializeField] private bool isWristPanel;
    
    void Update()
    {
        if (!Networking.LocalPlayer.IsValid()) return;
        
        if (isWristPanel)
        {
            // 手首追従
            VRCPlayerApi.TrackingData trackingData = 
                Networking.LocalPlayer.GetTrackingData(VRCPlayerApi.TrackingDataType.LeftHand);
            targetPanel.position = trackingData.position;
            targetPanel.rotation = trackingData.rotation;
        }
        else
        {
            // 頭部前方追従（メインパネル用）
            Transform head = Networking.LocalPlayer.GetTrackingData(
                VRCPlayerApi.TrackingDataType.Head).position;
            targetPanel.position = head.position + head.forward * followDistance;
            targetPanel.LookAt(head);
        }
    }
}
```

### フェーズ3: UIコンポーネントの実装

#### 3.1 手首パネルUI
1. **UI要素の配置**
   - 現在シーン表示用Text
   - クイックテレポートボタン
   - メインパネル展開ボタン

2. **展開ボタンスクリプト**
`Scripts/UI/PanelToggler.cs`:

```csharp
using UdonSharp;
using UnityEngine;
using UnityEngine.UI;

public class PanelToggler : UdonSharpBehaviour
{
    [SerializeField] private GameObject mainPanel;
    [SerializeField] private Button toggleButton;
    
    public override void Interact()
    {
        mainPanel.SetActive(!mainPanel.activeSelf);
    }
}
```

#### 3.2 メインパネルUI構築
要件定義1.2のパネル構成に従って以下を実装:

1. **シーン選択エリア**
   - 6つのシーンボタンをGridLayoutGroupで配置
   - 各ボタンにシーン番号とアイコンを設定

2. **ギミック制御エリア**
   - 現在選択中のシーンに応じて2つのギミックボタンを表示
   - Toggle方式で実装（ON/OFF切り替え可能）

3. **ロール選択エリア**
   - 要件定義2.3に基づくチェックボックス実装
   - デフォルト選択状態を設定

### フェーズ4: シーン管理システム

#### 4.1 シーンマネージャー実装
`Scripts/Core/SceneManager.cs`:

```csharp
using UdonSharp;
using UnityEngine;

[System.Serializable]
public class SceneData
{
    public string sceneName;
    public GameObject[] gimmicks;
    public Transform teleportPoint;
}

public class SceneManager : UdonSharpBehaviour
{
    [SerializeField] private SceneData[] scenes = new SceneData[6];
    [SerializeField] private int currentSceneIndex = 0;
    
    public void ChangeScene(int sceneIndex)
    {
        // 現在のシーンのギミックを無効化
        DisableCurrentGimmicks();
        
        // シーンインデックス更新
        currentSceneIndex = sceneIndex;
        
        // UIの更新
        UpdateSceneUI();
    }
    
    private void DisableCurrentGimmicks()
    {
        foreach (var gimmick in scenes[currentSceneIndex].gimmicks)
        {
            gimmick.SetActive(false);
        }
    }
}
```

### フェーズ5: ギミック制御システム

#### 5.1 基本ギミックインターフェース
要件定義2.2の4種類のギミックに対応:

```csharp
// Scripts/Gimmicks/BaseGimmick.cs
using UdonSharp;
using UnityEngine;

public abstract class BaseGimmick : UdonSharpBehaviour
{
    [SerializeField] protected string gimmickName;
    [SerializeField] protected bool isActive;
    
    public abstract void ActivateGimmick();
    public abstract void DeactivateGimmick();
    
    public void Toggle()
    {
        isActive = !isActive;
        if (isActive)
            ActivateGimmick();
        else
            DeactivateGimmick();
    }
}
```

#### 5.2 個別ギミック実装例

**音響系ギミック**:
```csharp
// Scripts/Gimmicks/AudioGimmick.cs
public class AudioGimmick : BaseGimmick
{
    [SerializeField] private AudioSource audioSource;
    [SerializeField] private AudioClip[] clips;
    [SerializeField] private bool is3DSound;
    
    public override void ActivateGimmick()
    {
        if (is3DSound)
        {
            // VRC Spatial Audio設定
            audioSource.spatialBlend = 1.0f;
        }
        audioSource.Play();
    }
    
    public override void DeactivateGimmick()
    {
        audioSource.Stop();
    }
}
```

### フェーズ6: テレポート機能

#### 6.1 ロール別テレポート実装
```csharp
// Scripts/Core/TeleportManager.cs
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

public class TeleportManager : UdonSharpBehaviour
{
    [SerializeField] private bool teleportParticipants = true;
    [SerializeField] private bool teleportOpponents = true;
    [SerializeField] private bool teleportSpectators = false;
    
    public void ExecuteTeleport(Transform destination)
    {
        VRCPlayerApi[] players = new VRCPlayerApi[80];
        int playerCount = VRCPlayerApi.GetPlayers(players);
        
        for (int i = 0; i < playerCount; i++)
        {
            if (ShouldTeleportPlayer(players[i]))
            {
                players[i].TeleportTo(destination.position, 
                                     destination.rotation);
            }
        }
    }
    
    private bool ShouldTeleportPlayer(VRCPlayerApi player)
    {
        // ロール判定ロジック実装
        // 実際の実装では、プレイヤーのカスタムプロパティなどで
        // ロールを管理する必要があります
        return true;
    }
}
```

### フェーズ7: 最適化とQuest対応

#### 7.1 パフォーマンス最適化
1. **UI最適化**
   - 使用していないパネルの非アクティブ化
   - UIのバッチング有効化
   - 透明度の使用を最小限に

2. **スクリプト最適化**
   - Update()の使用を最小限に
   - 必要な時のみ処理を実行

#### 7.2 Quest対応チェックリスト
- [ ] ポリゴン数の確認（50k以下推奨）
- [ ] テクスチャサイズの最適化（512×512以下）
- [ ] リアルタイムライトの削減
- [ ] パーティクル数の制限

### フェーズ8: テストとデバッグ

#### 8.1 単体テスト項目
1. **GM認証テスト**
   - GM以外のプレイヤーでパネルが表示されないこと
   - GMプレイヤーで正常に表示されること

2. **UI動作テスト**
   - 手首パネルの追従動作
   - メインパネルの展開/格納
   - 各ボタンの反応

3. **ギミック動作テスト**
   - 各ギミックのON/OFF切り替え
   - シーン変更時の状態リセット

#### 8.2 統合テスト項目
1. **マルチプレイヤーテスト**
   - テレポート機能の動作確認
   - ギミックの同期確認

2. **パフォーマンステスト**
   - FPS測定（要件5.2準拠）
   - メモリ使用量確認

### フェーズ9: デプロイメント

#### 9.1 ビルド前チェックリスト
- [ ] すべてのPrefabが正しく設定されている
- [ ] Lightmapのベイク完了
- [ ] コライダーの最適化
- [ ] 不要なアセットの削除

#### 9.2 VRChatアップロード
1. VRChat SDK Control Panelを開く
2. World Descriptorの設定確認
3. Build & Uploadを実行

## トラブルシューティング

### よくある問題と解決策

1. **パネルが表示されない**
   - GM認証の設定を確認
   - Canvasの設定を確認

2. **ボタンが反応しない**
   - VRC_UiShapeの設定を確認
   - コライダーの有無を確認

3. **パフォーマンスが悪い**
   - プロファイラーで負荷の高い処理を特定
   - 最適化チェックリストを再確認

## 次のステップ

### 拡張機能の実装
1. **カスタムギミックの追加**
   - BaseGimmickを継承して新規ギミック作成
   - SceneDataに追加

2. **設定ファイル対応**
   - ScriptableObjectでシーン設定を外部化
   - ランタイムでの設定変更対応

3. **高度な演出機能**
   - タイムライン連携
   - 複数ギミックの同期実行

## 参考リソース
- [VRChat Documentation](https://docs.vrchat.com/)
- [UdonSharp Documentation](https://udonsharp.docs.vrchat.com/)
- [Unity uGUI Documentation](https://docs.unity3d.com/Manual/UISystem.html)