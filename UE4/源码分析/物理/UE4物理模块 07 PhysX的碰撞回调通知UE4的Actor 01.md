# UE4物理模块 07 PhysX的碰撞回调通知UE4的Actor 01
## 回调函数的原理
引擎在`FPhysXSimEventCallback`(用于通知引擎关于各种碰撞事件的,回调事件,继承PxSimulationEventCallback)  
里面定义接口,捕获physx的事件,如onContact,onTrigger  

参数pairHeader中存有两个碰撞的actor,这里还是physx层面的actor,UE4引擎对应的actor保存在物理actor的userData里  

得到两个actor以及物理材质,碰撞点之类的数据后,碰撞信息会放在一个数组中  
`TArray<FCollisionNotifyInfo> FPhysScene_PhysX::PendingCollisionData.PendingCollisionNotifies`  
这里还是异步线程,不能直接调用主线程接口  

在物理结束后的`FPhysScene_PhysX::EndFrame`里,调用`FPhysScene_PhysX::DispatchPhysNotifications_AssumesLocked()`  
遍历PendingCollisionNotifies,获取里面的actor,调用`AActor::DispatchPhysicsCollisionHit`  
就可以把碰撞事件分发到引擎Actor中  

## `FPhysXSimEventCallback::onContact`
physx碰撞事件的接口,将碰撞的两个actor的碰撞信息存入PendingCollisionNotifies数组中  

```
void FPhysXSimEventCallback::onContact(
	const PxContactPairHeader& PairHeader, 
	const PxContactPair* Pairs, 
	PxU32 NumPairs)
{
	const PxActor* PActor0 = PairHeader.actors[0];
	const PxActor* PActor1 = PairHeader.actors[1];

	const PxRigidBody* PRigidBody0 = PActor0->is<PxRigidBody>();
	const PxRigidBody* PRigidBody1 = PActor1->is<PxRigidBody>();

	const FBodyInstance* BodyInst0 = FPhysxUserData::Get<FBodyInstance>(PActor0->userData);
	const FBodyInstance* BodyInst1 = FPhysxUserData::Get<FBodyInstance>(PActor1->userData);

	// TArray<FCollisionNotifyInfo> FPhysScene_PhysX::PendingCollisionData.PendingCollisionNotifies
	// 碰撞通知数组,等待物理引擎运行结束后执行
	TArray<FCollisionNotifyInfo>& PendingCollisionNotifies = OwningScene->GetPendingCollisionNotifies();

	// 记录添加前的索引
	uint32 PreAddingCollisionNotify = PendingCollisionNotifies.Num() - 1;
	// 向碰撞通知数组中添加新的碰撞信息数组,并返回添加的索引数组
	TArray<int32> PairNotifyMapping = AddCollisionNotifyInfo(BodyInst0, BodyInst1, Pairs, NumPairs, PendingCollisionNotifies);

	// 遍历接触点,更新物理材质,冲力等
	for(uint32 PairIdx=0; PairIdx<NumPairs; PairIdx++)
	{
		int32 NotifyIdx = PairNotifyMapping[PairIdx]; // 获取新添加的碰撞信息索引
		if (NotifyIdx == -1)
		{
			continue;
		}

		FCollisionNotifyInfo * NotifyInfo = &PendingCollisionNotifies[NotifyIdx];
		FCollisionImpactData* ImpactInfo = &(NotifyInfo->RigidCollisionData);

		const PxContactPair* Pair = Pairs + PairIdx;

		// 有些代码Shape0,Shape1类似,这里只写Shape0

		const PxShape* Shape0 = Pair->shapes[0];

		// 获取材质
		PxMaterial* Material0 = nullptr;
		UPhysicalMaterial* PhysMat0  = nullptr;
		// 如果我们有简单的几何或只有一种材质,在这里设置它,否则按面做
		if(Shape0->getNbMaterials() == 1)
		{
			Shape0->getMaterials(&Material0, 1);
			PhysMat0 = Material0 ? FPhysxUserData::Get<UPhysicalMaterial>(Material0->userData) : nullptr;
		}

		// 遍历接触点
		PxContactPairPoint ContactPointBuffer[16];
		// 从流中提取接触点,并以方便的格式存储
		int32 NumContactPoints = Pair->extractContacts(ContactPointBuffer, 16);
		for(int32 PointIdx=0; PointIdx<NumContactPoints; PointIdx++)
		{
			const PxContactPairPoint& Point = ContactPointBuffer[PointIdx];

			// 在法线上投影冲力
			const PxVec3 NormalImpulse = Point.impulse.dot(Point.normal) * Point.normal;
			ImpactInfo->TotalNormalImpulse += P2UVector(NormalImpulse);

			// 按面获取材质,这里是复杂的几何或多材质
			if(!Material0)
			{
				if(PxMaterial* Material0PerFace = Shape0->getMaterialFromInternalFaceIndex(Point.internalFaceIndex0))
				{
					PhysMat0 = FPhysxUserData::Get<UPhysicalMaterial>(Material0PerFace->userData);
				}
			}

			// 在每个碰撞pair中,添加这个pair的所有point的碰撞信息
			new(ImpactInfo->ContactInfos) FRigidBodyContactInfo(
				P2UVector(Point.position), 
				P2UVector(Point.normal), 
				-1.f * Point.separation, 
				PhysMat0, 
				PhysMat1);
		}
	}

	for (int32 NotifyIdx = PreAddingCollisionNotify + 1; NotifyIdx < PendingCollisionNotifies.Num(); NotifyIdx++)
	{
		FCollisionNotifyInfo * NotifyInfo = &PendingCollisionNotifies[NotifyIdx];
		FCollisionImpactData* ImpactInfo = &(NotifyInfo->RigidCollisionData);
		// 移除没有产生力的pair
		if (ImpactInfo->TotalNormalImpulse.SizeSquared() < KINDA_SMALL_NUMBER)
		{
			PendingCollisionNotifies.RemoveAt(NotifyIdx);
			NotifyIdx--;
		}
	}
}
```

+ 碰撞信息结构体  
```
// 物理引擎运行结束后,等待执行的碰撞通知信息
struct ENGINE_API FCollisionNotifyInfo
{
	bool bCallEvent0; // 是否调用通知事件
	bool bCallEvent1;
	FRigidBodyCollisionInfo Info0;
	FRigidBodyCollisionInfo Info1;
	FCollisionImpactData RigidCollisionData;
}

struct ENGINE_API FRigidBodyCollisionInfo
{
	TWeakObjectPtr<AActor> Actor;
	TWeakObjectPtr<UPrimitiveComponent> Component;
	int32 BodyIndex;
	FName BoneName;
}

struct ENGINE_API FCollisionImpactData
{
	TArray<struct FRigidBodyContactInfo> ContactInfos;
}

struct ENGINE_API FRigidBodyContactInfo
{
	FVector ContactPosition;
	FVector ContactNormal;
	class UPhysicalMaterial* PhysMaterial[2];
}
```

+ 添加碰撞信息  
```
// 我的理解:两个actor碰撞,这里是Body0,Body1参数,每个FBodyInstance都有[子形状-子FBodyInstance]的映射,这里就是Pairs参数
// 对于每个子形状的碰撞信息,都在参数PendingNotifyInfos中添加一条新的FCollisionNotifyInfo
// 最后返回所有新添加的PendingNotifyInfos的索引
TArray<int32> AddCollisionNotifyInfo(
	const FBodyInstance* Body0, // 我理解的是,actor的整体FBodyInstance
	const FBodyInstance* Body1, 
	const physx::PxContactPair * Pairs, // 我理解的是,具体碰撞的子FBodyInstance的形状
	uint32 NumPairs, 
	TArray<FCollisionNotifyInfo> & PendingNotifyInfos)
{
	// PendingNotifyInfos中的索引
	TArray<int32> PairNotifyMapping;
	PairNotifyMapping.Empty(NumPairs);

	// [body0 - [body1 - 碰撞信息在PendingNotifyInfos数组中的索引]] 的映射
	TMap<const FBodyInstance*, TMap<const FBodyInstance*, int32> > BodyPairNotifyMap;
	for(uint32 PairIdx = 0; PairIdx < NumPairs; ++PairIdx)
	{
		const PxContactPair* Pair = Pairs + PairIdx;
		PairNotifyMapping.Add(-1); // 初始值为-1,因为可以不记录碰撞

		const PxShape* Shape0 = Pair->shapes[0];
		const PxShape* Shape1 = Pair->shapes[1];

		// 大概是FBodyInstance里面有,子形状-子FBodyInstance的映射ShapeToBodiesMap,根据形状,从中查找子FBodyInstance
		const FBodyInstance* SubBody0 = FPhysicsInterface_PhysX::ShapeToOriginalBodyInstance(Body0, Shape0);
		const FBodyInstance* SubBody1 = FPhysicsInterface_PhysX::ShapeToOriginalBodyInstance(Body1, Shape1);

		TMap<const FBodyInstance *, int32> & SubBodyNotifyMap = BodyPairNotifyMap.FindOrAdd(SubBody0);
		int32* NotifyInfoIndex = SubBodyNotifyMap.Find(SubBody1);

		if(NotifyInfoIndex == NULL)
		{
			// 在传入的参数PendingNotifyInfos数组末尾添加新的FCollisionNotifyInfo
			FCollisionNotifyInfo * NotifyInfo = new (PendingNotifyInfos) FCollisionNotifyInfo;
			NotifyInfo->Info0.SetFrom(SubBody0);
			NotifyInfo->Info1.SetFrom(SubBody1);

			NotifyInfoIndex = &SubBodyNotifyMap.Add(SubBody0, PendingNotifyInfos.Num() - 1);
		}

		PairNotifyMapping[PairIdx] = *NotifyInfoIndex;
	}

	return PairNotifyMapping;
}
```