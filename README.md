# TweenLib

## Introduction

**TweenLib** is a tweening library for Unity ECS that enables field-level tweening of `IComponentData`. Instead of providing built-in tween logic, it allows you to define your own logic for full flexibility.

## Installation

To install, paste the following URL into Unity's **Package Manager**:

1. Open **Package Manager**.
2. Click the **+** button.
3. Select **"Add package from git URL..."**.
4. Enter: `https://github.com/hoangtongvu/com.darksun.tweenlib.git`

## How to use

### 1. Use predefined Tweener or declare your own

TweenLib provides several ready-to-use Tweeners in the [StandardTweeners](StandardTweeners) assembly:
- [TransformPositionTweener](StandardTweeners/TransformPositionTweener.cs)
- [TransformRotationTweener](StandardTweeners/TransformRotationTweener.cs)
- *(More coming soon...)*

Or, you can create your own Tweener with custom logic:
- The Tweener must be **partial struct**.
- The Tweener must implements `ITweener<Component, Target>`, where:
    - `Component` is the `IComponentData` that you want to tween on.
    - `Target` is the tween destination value.

**A Tweener example** ([TransformPositionTweener](StandardTweeners/TransformPositionTweener.cs)):
```cs
[BurstCompile]
public partial struct TransformPositionTweener : ITweener<LocalTransform, float3>
{
    [BurstCompile]
    public bool CanStop(in LocalTransform componentData, in float lifeTimeSecond, in float baseSpeed, in float3 target)
    {
        return math.all(math.abs(target - componentData.Position) < new float3(Configs.Epsilon));
    }

    [BurstCompile]
    public void Tween(ref LocalTransform componentData, in float baseSpeed, in float3 target)
    {
        componentData.Position =
            math.lerp(componentData.Position, target, baseSpeed * this.DeltaTime);
    }
}
```

### 2. Attach generated components to your entities

There are 2 components generated by source generator, named based on your Tweener name.
- `{tweenerName}_TweenData`
- `Can_{tweenerName}_TweenTag`

You can attach them into your entity using **Authorings** or **Systems**.

**CubeAuthoring example:**
```cs
public class CubeAuthoring : MonoBehaviour
{
    private class Baker : Baker<CubeAuthoring>
    {
        public override void Bake(CubeAuthoring authoring)
        {
            Entity entity = GetEntity(authoring, TransformUsageFlags.Dynamic);

            AddComponent<Can_TransformPositionTweener_TweenTag>(entity);
            SetComponentEnabled<Can_TransformPositionTweener_TweenTag>(entity, false);
            AddComponent<TransformPositionTweener_TweenData>(entity);
        }
    }
}
```

### 3. Use `TweenBuilder` to create your tween

To trigger tweening, use `TweenBuilder` in your systems, which can be accessed via `Tweener`.

**A TweenBuilder usage example:**
```cs
foreach (var (transformRef, moveStateRef, tweenDataRef, canTweenTag, moveStateChangedTag) in
    SystemAPI.Query<
        RefRO<LocalTransform>
        , RefRO<MoveStateICD>
        , RefRW<TransformPositionTweener_TweenData>
        , EnabledRefRW<Can_TransformPositionTweener_TweenTag>
        , EnabledRefRO<MoveStateChangedTag>>()
        .WithOptions(EntityQueryOptions.IgnoreComponentEnabledState))
{
    if (!moveStateChangedTag.ValueRO) continue;
    if (moveStateRef.ValueRO.Value != MoveState.Right) continue;

    const float rightX = 8f;
    const float baseSpeed = 2f;

    TransformPositionTweener.TweenBuilder.Create()
        .WithBaseSpeed(baseSpeed)
        .WithTarget(new float3(rightX, 0, 0))
        .Build(ref tweenDataRef.ValueRW, canTweenTag);

}
```

**Note**: Currently, **TweenLib** only support controling tween life time via `BaseSpeed`.

## Known issues

- **Multiple Interface Implementation:** If a Tweener implements multiple interfaces and ITweener is not the first one listed, the source generator may not work correctly.