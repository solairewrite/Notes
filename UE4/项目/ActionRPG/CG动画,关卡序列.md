# 关卡序列
## 类型
创建`LevelSequence`,放入关卡中变为`LevelSequenceActor`,可通过`GetAllActorsOfClass`获得 
``` 
class ALevelSequenceActor
{
    `ULevelSequencePlayer* SequencePlayer;`
}
```
```
class ULevelSequencePlayer
	: public UMovieSceneSequencePlayer
{
    UMovieSceneSequencePlayer::IsPlaying()

    UMovieSceneSequencePlayer::Play()

    UMovieSceneSequencePlayer::Stop()
}
```