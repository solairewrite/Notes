# AI寻路
.build.cs 中添加 "AIModule", "NavigationSystem"  

+ 寻路  
```
#include "Kismet/GameplayStatics.h"
#include "NavigationPath.h"
#include "NavigationSystem.h"

UNavigationPath* NavPath = UNavigationSystemV1::FindPathToActorSynchronously(this, GetActorLocation(), PlayerPawn);

if (NavPath && NavPath->PathPoints.Num() > 1)
{
    return NavPath->PathPoints[1];
}
```

+ 移动  
```
#include "Blueprint/AIBlueprintHelperLibrary.h"

UAIBlueprintHelperLibrary::SimpleMoveToActor(controller, player);
```

+ 停止移动  
`controller->StopMovement();`  
