+ Log显示类名(行号)  
```
#define MYCLASS (FString(__FUNCTION__).Left(FString(__FUNCTION__).Find(TEXT(":"))))

#define MYLINE  (FString::FromInt(__LINE__))

#define MYCLASSLINE (MYCLASS + "(" + MYLINE +")")

#define NORMALLOG(FormatString , ...) UE_LOG(LogTemp,Log,TEXT("%s: %s"), 	*MYCLASSLINE, *FString::Printf(TEXT(FormatString), ##__VA_ARGS__ ))
```