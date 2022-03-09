# UE4物理模块 08 PhysX的碰撞回调通知UE4的Actor 02
## PhysX的碰撞回调通知UE4的Actor
在物理结束后的`FPhysScene_PhysX::EndFrame`里,调用`FPhysScene_PhysX::DispatchPhysNotifications_AssumesLocked()`  
遍历PendingCollisionNotifies,获取里面的actor,调用`AActor::DispatchPhysicsCollisionHit`  
就可以把碰撞事件分发到引擎Actor中  

实测,PendingCollisionNotifies数组仅在`Simulate Physics`为true时才有数据  
即下面的逻辑都是在模拟物理时生效  
角色移动和Set Location并且勾选sweep走的是其它逻辑  

```
void FPhysScene_PhysX::EndFrame(ULineBatchComponent* InLineBatcher)
{
    // 执行碰撞通知事件
    DispatchPhysNotifications_AssumesLocked();
}

void FPhysScene_PhysX::DispatchPhysNotifications_AssumesLocked()
{
    TArray<FCollisionNotifyInfo>& PendingCollisionNotifies = GetPendingCollisionNotifies();

    // 触发队列中的碰撞通知
    for (int32 i = 0; i<PendingCollisionNotifies.Num(); i++)
    {
        FCollisionNotifyInfo& NotifyInfo = PendingCollisionNotifies[i];
        if (NotifyInfo.RigidCollisionData.ContactInfos.Num() > 0)
        {
            if (NotifyInfo.bCallEvent0 && NotifyInfo.IsValidForNotify() && NotifyInfo.Info0.Actor.IsValid())
            {
                NotifyInfo.Info0.Actor->DispatchPhysicsCollisionHit(NotifyInfo.Info0, NotifyInfo.Info1, NotifyInfo.RigidCollisionData);
            }

            // 需要再次检测IsValidForNotify,以防上面的调用改动了什么
            if (NotifyInfo.bCallEvent1 && NotifyInfo.IsValidForNotify() && NotifyInfo.Info1.Actor.IsValid())
            {
                // 交换物理材质0,物理材质1,法线反向
                NotifyInfo.RigidCollisionData.SwapContactOrders();
                NotifyInfo.Info1.Actor->DispatchPhysicsCollisionHit(NotifyInfo.Info1, NotifyInfo.Info0, NotifyInfo.RigidCollisionData);
            }
        }
    }
    PendingCollisionNotifies.Reset();
}
```

+ UE4 Actor触发碰撞事件  
```
void AActor::DispatchPhysicsCollisionHit(const FRigidBodyCollisionInfo& MyInfo, const FRigidBodyCollisionInfo& OtherInfo, const FCollisionImpactData& RigidCollisionData)
{
    // 暂时仅传入碰撞信息数组的第一个碰撞,可能会优化
    const FRigidBodyContactInfo& ContactInfo = RigidCollisionData.ContactInfos[0];

    // 根据physx的碰撞信息,填充UE4的碰撞结构
    FHitResult Result;
	Result.Location = Result.ImpactPoint = ContactInfo.ContactPosition;
	Result.Normal = Result.ImpactNormal = ContactInfo.ContactNormal;
	Result.PenetrationDepth = ContactInfo.ContactPenetration;
	Result.PhysMaterial = ContactInfo.PhysMaterial[1];
	Result.Actor = OtherInfo.Actor;
	Result.Component = OtherInfo.Component;
	Result.Item = OtherInfo.BodyIndex;
	Result.BoneName = OtherInfo.BoneName;
	Result.MyBoneName = MyInfo.BoneName;
	Result.bBlockingHit = true;

    // 当一个actor碰撞一个阻挡物体时触发
    // 可能是角色移动,Set Location并且勾选sweep,或模拟物理
    // 里面同时调用了蓝图事件Hit
	NotifyHit(MyInfo.Component.Get(), OtherInfo.Actor.Get(), OtherInfo.Component.Get(), true, Result.Location, Result.Normal, RigidCollisionData.TotalNormalImpulse, Result);
}
```

## 自定义physx的simulation回调
PxSceneDesc是physx的场景描述符,用于创建physx的场景  
`PxScene* PScene = GPhysXSDK->createScene(PSceneDesc);`  

通过PSceneDesc.filterShader实现自定义回调  
UE4给了一个接口GSimulationFilterShader,默认是空的  
如果没有设置就用physx默认的PhysXSimFilterShader  

```
// 用于过滤模拟碰撞的shader,可以在任何线程上调用
PxFilterFlags PhysXSimFilterShader(	
    PxFilterObjectAttributes attributes0, // 判断碰撞盒的属性,是否是kinematic或static
    PxFilterData filterData0, 
	PxFilterObjectAttributes attributes1, 
    PxFilterData filterData1,
	PxPairFlags& pairFlags, 
    const void* constantBlock, 
    PxU32 constantBlockSize );

// 传入collision filtering shader and/or callback的简单结构,提供碰撞pair的额外信息
typedef PxU32 PxFilterObjectAttributes;

// 用户定义的数据,传入collision filtering shader and/or callback
// word属性判断相互接触的逻辑(是否启用CCD,碰撞解算,只是trigger回调)
struct PxFilterData
{
    PxU32 word0;
	PxU32 word1;
	PxU32 word2;
	PxU32 word3;
}
```

+ 根据场景描述符创建物理场景
```
void FPhysScene_PhysX::InitPhysScene(const AWorldSettings* Settings)
{
    // 在循环中包含场景描述符,这样就可以根据场景类型修改它
    PxSceneDesc PSceneDesc(GPhysXSDK->getTolerancesScale());
	PSceneDesc.cpuDispatcher = CPUDispatcher;

	FPhysSceneShaderInfo PhysSceneShaderInfo;
	PhysSceneShaderInfo.PhysScene = this;
	PSceneDesc.filterShaderData = &PhysSceneShaderInfo;
	PSceneDesc.filterShaderDataSize = sizeof(PhysSceneShaderInfo);

	PSceneDesc.filterShader = GSimulationFilterShader ? GSimulationFilterShader : PhysXSimFilterShader;
	PSceneDesc.simulationEventCallback = SimEventCallback;

    // PSceneDesc.flags里面有大量根据UPhysicsSettings的配置
    // 如是否使用CCD,MBP等,这里只列出部分配置
    if (UPhysicsSettings::Get()->bDisableCCD == false)
	{
		PSceneDesc.flags |= PxSceneFlag::eENABLE_CCD;
	}

    // 创建physx的场景
    PxScene* PScene = GPhysXSDK->createScene(PSceneDesc);

    // physx的粗划分算法分为两种,默认SAP(sweep and prune),MBP(multi box pruning)
    // 大多数actor不移动时SAP效率高,有大量的actor加入/删除/移动时,MBP效率高

    // apex用来做布料模拟和破坏效果,后面可能弃用

    // 整合PVD性能工具

    // 物理初始化结束
}
```