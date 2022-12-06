# 回放

UE4的回放，我理解为是基于网络同步的基础，创建一个DemoNetConnection以网络同步的形式录制场景中的网络对象（在LowLevelSend录制Packets），并记录Packet生成的时间，进行序列化。
播放时再解析成Packets，Tick驱动取出对应回放时间Packets，调用UNetConnection::ReceivedRawPacket对当前Packets像收到的网络包一样进行处理。

## 回放系统架构概要

UReplay：一个全局的回放子系统，用于封装核心接口并暴露给上层调用。（注：Subsystem类似C++中的单例类）
DemoNetdriver：继承自NetDriver，专门用于宏观地控制回放系统的录制与播放。
Demonetconnection：继承自NetConnection，可以自定义实现回放数据的发送位置。
FReplayHelper：封装一些回放处理数据的接口，用于将回放逻辑与DemoNetDriver进行解耦。
XXXNetworkReplayStreamer：回放序列化数据的存储类，根据不同的存储方式有不同的具体实现。

Http：把回放的数据定时的通过Http发送到一个指定url的服务器上
InMemory：不断的将回放数据写到内存里面，可以随时快速地取出
LocalFile:写到本地指定目录的文件里面，维护了一个FQueuedLocalFileRequest队列不停地按顺序处理数据的写入和加载 NetWork:各种基类接口、基类工厂
Null：早期默认的存储方式，通过Json写到本地文件里面，但是效率比较低（已废弃）
SavGame：LocalFile的前身，现在已经完全继承并使用LocalFile的实现

## 回放数据

录制的数据实际上就是网络同步中的一个个Packet
数据流的类型：
Checkpoint：存档点，即一个完整的世界快照（类似单机游戏中的存档），通过这个快照可以完全的回复当时的游戏状态。每隔一段时间（比如30s）存储一个checkpoint。
Stream：一段连续时间的数据流，存储着从上一个Checkpoint到当前的所有序列化录制数据

回放数据在内存中存在的形式：

``` c++
struct FQueuedDemoPacket
{
    TArray<uint8> Data;//Packet的数据
    int32 SizeBits;//Packet大小
    uint32 SeenLevelIndex;//和该Packet关联的LevelIndex
}
class FReplayHelper
{
    //当前的时间DemoCurrentTime也会被序列化到Archive里面  
    TArray<FQueuedDemoPacket> QueuedDemoPackets;
    TArray<FQueuedDemoPacket> QueuedCheckpointPackets;
}
class FLocalFileStreamFArchive : public FArchive
{
    //保存所有序列化的数据
    TArray<uint8>    Buffer;
    int32            Pos;
    bool            bAtEndOfReplay;
}
//加载回放时保存完整回放数据
struct FLocalFileReplayInfo
{

}
```

``` c++
//sendbuffer（packet）塞入QueuedPackets
void UDemoNetConnection::LowLevelSend(void* Data, int32 CountBits, FOutPacketTraits& Traits)
{
    TArray<FQueuedDemoPacket>& QueuedPackets = (ResendAllDataState != EResendAllDataState::None) ? DemoDriver->ReplayHelper.QueuedCheckpointPackets : DemoDriver->ReplayHelper.QueuedDemoPackets;
    int32 NewIndex = QueuedPackets.Emplace((uint8*)Data, CountBits, Traits);
}

//写入Archive
void FReplayHelper::WriteDemoFrame(UNetConnection* Connection, FArchive& Ar, TArray<FQueuedDemoPacket>& QueuedPackets, float FrameTime, EWriteDemoFrameFlags Flags)
{
    Ar << CurrentLevelIndex;
    Ar << FrameTime;

}

//读取Archive
bool FReplayHelper::ReadDemoFrame(UNetConnection* Connection, FArchive& Ar, TArray<FPlaybackPacket>& InPlaybackPackets, const bool bForLevelFastForward, const FArchivePos MaxArchiveReadPos, float* OutTime)
{
    Ar << ReadCurrentLevelIndex;
    *OutTime = TimeSeconds;
    //读取StreamingLevels的序列化并加载

    //读取Archive到QueuedPackets
    FPlaybackPacket ScratchPacket;
    ScratchPacket.TimeSeconds = TimeSeconds;//记录回放时间戳
    ScratchPacket.LevelIndex = ReadCurrentLevelIndex;
    switch (ReadPacket(Ar, ScratchPacket.Data, ReadPacketMode))
    {
    case EReadPacketState::Success:
    {
        InPlaybackPackets.Emplace(MoveTemp(ScratchPacket));
        ScratchPacket.Data = TArray<uint8>();
        break;
    }
    }
}
```

## 录制Stream

录制入口

``` c++

void UDemoNetDriver::TickDemoRecordFrame(float DeltaSeconds)
{
    // 1. 获取所有Replicated的Actor（NumActiveObjects）
    FNetworkObjectList& NetObjectList = GetNetworkObjectList();
    const FNetworkObjectList::FNetworkObjectSet& ActiveObjectSet = NetObjectList.GetActiveObjects();
    const int32 NumActiveObjects = ActiveObjectSet.Num();

    // 2. 确定ViewTarget中心坐标用于优先级排序
    APlayerController* Viewer = ViewerOverride.Get();
    AActor* ViewTarget = Viewer ? Viewer->GetViewTarget() : nullptr;
    if (ViewTarget)
    {
        ViewLocation = ViewTarget->GetActorLocation();
        ViewDirection = ViewTarget->GetActorRotation().Vector();
    }

    //3.遍历所有Replicated的Actor
    for (const TSharedPtr<FNetworkObjectInfo>& ObjectInfo : ActiveObjectSet)
    {
        FNetworkObjectInfo* ActorInfo = ObjectInfo.Get();
        //是否到达录制时间
        if (GetDemoCurrentTime() > ActorInfo->NextUpdateTime)
        {
            //移除休眠的、不相关的Actor，添加到ActorsToRemove

            //添加到PrioritizedActors,
            //struct FActorPriority{int32 Priority; FNetworkObjectInfo* ActorInfo; class UActorChannel* Channel;}
            //struct FNetworkObjectInfo{AActor* Actor; double LastNetReplicateTime;}
            PrioritizedActors.Add(DemoActorPriority);
        }
    }
    //4.排序Actor
    PrioritizedActors.Sort([](const FDemoActorPriority& A, const FDemoActorPriority& B) { return B.ActorPriority.Priority < A.ActorPriority.Priority; });
    //5.复制Actor，调用到UActorChannel::ReplicateActor，
    //UNetConnection::SendRawBunch会把rawbunch的数据都放进SendBuffer（相当于Packet）
    //UDemoNetConnection::LowLevelSend会把SendBuffer（Packet）的数据放进QueuedPackets
    ReplicatePrioritizedActors(PrioritizedActors.GetData(), PrioritizedActors.Num(), Params);
    //6.把QueuedDemoPackets写入StreamAr（FLocalFileStreamFArchive）
    WriteDemoFrameFromQueuedDemoPackets(*FileAr, ReplayHelper.QueuedDemoPackets, GetDemoCurrentTime(), EWriteDemoFrameFlags::None);
}
```

## 录制CheckPoint

DemoNetDriver会每帧检查是否需要保存TickCheckpoint。
CheckpointSaveContext看做是一个状态机，通过状态切换分帧做保存checkpoint的步骤。

``` c++
//阶段枚举
enum class ECheckpointSaveState
{
    Idle,//检测是否需要保存，需要则切到ProcessCheckpointActors状态开始保存checkpoint流程
    ProcessCheckpointActors,//遍历并序列化所有Actor的相关数据
    SerializeDeletedStartupActors,//处理那些被删掉的对象
    CacheNetGuids,//缓存所有同步Actor的GUID
    SerializeGuidCache,//序列化所有同步Actor的GUID
    SerializeNetFieldExportGroupMap,//导出所有同步属性基本信息FieldExport GroupMap，用于播放时准确且能兼容地接收这些属性
    SerializeDemoFrameFromQueuedDemoPackets,//通过WriteDemoFrame把所有QueuedPackets写到Checkpoint Archive里面
    Finalize,//调用FlushCheckpoint把当前的StreamArchive和Checkpoint Archive写到目标位置（内存、本地磁盘、Http请求等）
};

//检测是否需要保存checkpoint，启动保存后驱动保存流程进行
void UDemoNetDriver::TickDemoRecord(float DeltaSeconds)
{
    //设置当前replay的时间
    SetDemoCurrentTime(GetDemoCurrentTime() + FReplayHelper::GetClampedDeltaSeconds(World, DeltaSeconds));

    //处在保存checkpoint中的某个阶段，继续分帧处理快照的录制
    if (ReplayHelper.GetCheckpointSaveState() != FReplayHelper::ECheckpointSaveState::Idle)
    {
        ReplayHelper.TickCheckpoint(ClientConnections[0]);
    }
    else
    {
        //录制这一帧的Stream，QueuedDemoPackets的数据写到ReplayStreamer里面
        TickDemoRecordFrame(DeltaSeconds);
        //到时间开始检测是否需要保存
        if (ReplayHelper.ShouldSaveCheckpoint())
        {
            ReplayHelper.SaveCheckpoint(ClientConnections[0]);
        }
    }
}

//收集checkpoint需要同步的actor，并进入保存流程
void FReplayHelper::SaveCheckpoint(UNetConnection* Connection)
{
    //排序NetworkObjectList所有网络同步的Object
    for (const TSharedPtr<FNetworkObjectInfo>& ObjectInfo : NetworkObjectList.GetAllObjects())
    {
        AActor* Actor = ObjectInfo.Get()->Actor;
        if (!bDeltaCheckpoint || ObjectInfo->bDirtyForReplay)
        {
            //判断是否在录制Connection中存在绑定了该Actor的Channel
            CheckpointSaveContext.PendingCheckpointActors.Add({ Actor, -1 });
        }
    }
    //进入第一个保存阶段
    CheckpointSaveContext.CheckpointSaveState = ECheckpointSaveState::ProcessCheckpointActors;
    CheckpointSaveContext.TotalCheckpointActors = CheckpointSaveContext.PendingCheckpointActors.Num();
}

//分帧处理快照的录制
void FReplayHelper::TickCheckpoint()
{
    //checkpoint分帧保存，状态控制
    while (bExecuteNextState && (CheckpointSaveContext.CheckpointSaveState != ECheckpointSaveState::Finalize) && !(Params.CheckpointMaxUploadTimePerFrame > 0 && CurrentTime - Params.StartCheckpointTime > Params.CheckpointMaxUploadTimePerFrame))
    {
        switch (CheckpointSaveContext.CheckpointSaveState)
        {

        //1.遍历并序列化所有Actor的相关数据
        case ECheckpointSaveState::ProcessCheckpointActors:
        {
            //复制所有PendingCheckpointActors中的actor，最终序列化到QueuedCheckpointPackets中
            do
            {
                const FPendingCheckPointActor Current = CheckpointSaveContext.PendingCheckpointActors.Pop();
                AActor* Actor = Current.Actor.Get();
                bContinue = ReplicateCheckpointActor(Actor, Connection, Params);
                // 调用Channel->ReplicateActor()
                // 同步actor以及component的属性和rpc
                // WroteSomethingImportant |= ActorReplicator->ReplicateProperties(Bunch, RepFlags);
                // WroteSomethingImportant |= Actor->ReplicateSubobjects(this, &Bunch, &RepFlags);
                // 如有修改调UChannel::SendBunch->UNetConnection::SendRawBunch->UDemoNetConnection::LowLevelSend
                // 最终序列化到QueuedCheckpointPackets中
                // FPacketIdRange PacketRange = SendBunch( &Bunch, 1 );
            } while (--NumActorsToReplicate && bContinue);
        }break;
        //2.处理被删掉的对象，写进
        case ECheckpointSaveState::SerializeDeletedStartupActors:
        {
            //
            *CheckpointArchive << CurrentLevelIndex;
            //destroy的actors会存在DeletedNetStartupActors，全部放入CheckpointArchive
            WriteDeletedStartupActors(Connection, *CheckpointArchive, DeletedNetStartupActors);
        }break;
        //3.缓存所有同步Actor的GUID
        case ECheckpointSaveState::CacheNetGuids:
        {
            //FReplayHelper::CacheNetGuids
            for (auto It = Connection->Driver->GuidCache->ObjectLookup.CreateIterator(); It; ++It)
            {
                FNetworkGUID& NetworkGUID = It.Key();
                FNetGuidCacheObject& CacheObject = It.Value();
                CheckpointSaveContext.NetGuidCacheSnapshot.Add({ NetworkGUID, CacheObject });
                CacheObject.bDirtyForReplay = false;
            }
        }break;
        //4.序列化上一步缓存的Actor的GUID(FReplayHelper::SerializeGuidCache)
        case ECheckpointSaveState::SerializeGuidCache:
        {
        }
        //5.导出所有同步属性基本信息FieldExport GroupMap，用于播放时准确且能兼容地接收这些属性
        case ECheckpointSaveState::SerializeNetFieldExportGroupMap:
        {
            PackageMapClient->SerializeNetFieldExportGroupMap(*CheckpointArchive);
        }
        //6.通过WriteDemoFrame把所有QueuedPackets写到Checkpoint Archive里面
        case ECheckpointSaveState::SerializeDemoFrameFromQueuedDemoPackets:
        {
            WriteDemoFrame(Connection, *CheckpointArchive, QueuedCheckpointPackets, static_cast<float>(LastCheckpointTime), EWriteDemoFrameFlags::SkipGameSpecific);
        }
        }
    }

    *CheckpointArchive << CurrentLevelIndex;
    *CheckpointArchive << DeletedNetStartupActors;
    SerializeGuidCache( GuidCache, CheckpointArchive );
    //调用FlushCheckpoint把当前的StreamArchive和Checkpoint Archive写到目标位置（内存、本地磁盘、Http请求等）
    WriteDemoFrameFromQueuedDemoPackets( *CheckpointArchive, ClientConnections[0]->QueuedCheckpointPackets);
}
```

## 播放Stream

``` c++
void UDemoNetDriver::TickDemoPlayback(float DeltaSeconds)
{

    //1.根据当前的时间，读取5s以内的StreamArchive里面的数据，缓存到PlaybackPackets数组里面(FReplayHelper::ReadDemoFrame)
    while (ConditionallyReadDemoFrameIntoPlaybackPackets(*GetReplayStreamer()->GetStreamingArchive()));

    //2.如果目标时间比快照的时间要大，需要把这段时间差的数据包全部读出来并进行处理（UDemoNetDriver::ProcessPacket调用到DemoNetConnection的ReceivedRawPacket）
    while (ConditionallyProcessPlaybackPackets())
    {
        DemoFrameNum++;
        ReplayHelper.DemoFrameNum++;
    }

    //3.删除已经执行的Packets
    if (PlaybackPacketIndex > 0)
    {
        LastProcessedPacketTime = PlaybackPackets[PlaybackPacketIndex - 1].TimeSeconds;
        PlaybackPackets.RemoveAt(0, PlaybackPacketIndex);
        PlaybackPacketIndex = 0;
    }
    //4.处理快进等操作，
    FinalizeFastForward(FastForwardStartSeconds);
}

bool FReplayHelper::ReadDemoFrame(UNetConnection* Connection, FArchive& Ar, TArray<FPlaybackPacket>& InPlaybackPackets, const bool bForLevelFastForward, const FArchivePos MaxArchiveReadPos, float* OutTime)
{
    Ar << TimeSeconds;
    //关卡流加载处理
    for (uint32 i = 0; i < NumStreamingLevels; ++i)
    {
        Ar << PackageName;
        Ar << PackageNameToLoad;
        Ar << LevelTransform;
        ULevelStreamingDynamic* StreamingLevel = NewObject<ULevelStreamingDynamic>(World.Get(), NAME_None, RF_NoFlags, nullptr);
        StreamingLevel->LevelTransform = LevelTransform;
        StreamingLevel->PackageNameToLoad = FName(*PackageNameToLoad);
        StreamingLevel->SetWorldAssetByPackageName(FName(*PackageName));
        World->AddStreamingLevel(StreamingLevel);
    }

    FPlaybackPacket ScratchPacket;
    ScratchPacket.TimeSeconds = TimeSeconds;//记录跑时间戳
    ScratchPacket.LevelIndex = ReadCurrentLevelIndex;
    while ((MaxArchiveReadPos == 0) || (Ar.Tell() < MaxArchiveReadPos))
    {
        //读Ar到ScratchPacket
        switch (ReadPacket(Ar, ScratchPacket.Data, ReadPacketMode))
        {
            case EReadPacketState::Success:
            {
                InPlaybackPackets.Emplace(MoveTemp(ScratchPacket));
                ScratchPacket.Data = TArray<uint8>();
                break;
            }
        }
    }
}

bool UDemoNetDriver::ConditionallyProcessPlaybackPackets()
{
    //取出当前执行到的PlaybackPacket
    const FPlaybackPacket& CurPacket = PlaybackPackets[PlaybackPacketIndex];
    //执行的packet已经赶上当前的时间
    if (GetDemoCurrentTime() < CurPacket.TimeSeconds) return false
    ++PlaybackPacketIndex;
    //调用ServerConnection->ReceivedRawPacket
    return ProcessPacket(CurPacket);
}
```

## 加载checkpoint

``` c++
class FGotoTimeInSecondsTask : public FQueuedReplayTask
{
    
    virtual void StartTask() override
    {
        OldTimeInSeconds = Driver->GetDemoCurrentTime();
        //设置当前的目标时间
        Driver->SetDemoCurrentTime(TimeInSeconds);
        //调用ReplayStreamer的GotoTimeInMS去查找要回放的数据流位置
        Driver->GetReplayStreamer()->GotoTimeInMS(Driver->GetDemoCurrentTimeInMS(), FGotoCallback::CreateSP(this, &FGotoTimeInSecondsTask::CheckpointReady), CheckpointType);
        //暂停回放的逻辑
        Driver->PauseChannels(true);
    }
}

bool UDemoNetDriver::LoadCheckpoint(const FGotoResult& GotoResult)
{
    //1.反序列化Level的Index
    *GotoCheckpointArchive << LevelForCheckpoint;
    if (LevelForCheckpoint != GetCurrentLevelIndex())
    {
        //如果当前的Level与Index标记的Level不同，需要把Actor删掉然后无缝加载目标的Level
        for (FActorIterator It(World); It; ++It)
            World->DestroyActor(*It, true);
    }

    //2.把SpectatorController、ViewTarget、bAlwaysRelevant的actor的GUID添加到NonQueuedGUIDsForScrubbing中，其他要同步的actor会进入队列慢慢处理。
    AddNonQueuedActorForScrubbing(SpectatorController);
    AddNonQueuedActorForScrubbing(SpectatorController->GetViewTarget());

    for (FActorIterator It(World); It; ++It)
    {
        if (It->bAlwaysRelevant)
        {
            AddNonQueuedActorForScrubbing(*It);
        }

        //3.保留被SepctatorController引用的Actor，以及NetStartUpActor，添加到KeepAliveActors，其余删除
    }
    //4.删除ServerConnection中KeepAliveActors的Channel的actor引用，防止删除ServerConnection的时候一起删除
    for (int32 i = ServerConnection->OpenChannels.Num() - 1; i >= 0; i--)
    {
        UActorChannel* ActorChannel = Cast<UActorChannel>(ServerConnection->OpenChannels[i]);
        if (ActorChannel != nullptr && KeepAliveActors.Contains(ActorChannel->Actor))
            ActorChannel->Actor = nullptr;
    }
    if (ServerConnection->OwningActor == SpectatorController)
        ServerConnection->OwningActor = nullptr;
    ServerConnection->Close();
    ServerConnection->CleanUp();

    ServerConnection = NewObject<UNetConnection>(GetTransientPackage(), UDemoNetConnection::StaticClass());
    ServerConnection->InitConnection( this, USOCK_Pending, ConnectURL, 1000000 );

    //反序列化Checkpoint的时间、关卡信息等内容
    //准备好回访数据，将CheckpointArchive里面的回放数据读取到FPlaybackPacket数组
    ReadDemoFrameIntoPlaybackPackets(*GotoCheckpointArchive);
    //获取最后一个数据包的时间用作当前的回放时间，然后根据跳跃的时长设置最终的目标时间
    SetDemoCurrentTime((PlaybackPackets.Num() > 0) ? PlaybackPackets.Last().TimeSeconds : 0.0f);
    //解析FPlaybackPacket，反序列所有的Actor数据
    ProcessAllPlaybackPackets();
}
```

## 开始录像

``` c++
 UGameInstance::StartRecordingReplay
    UDemoNetDriver::InitListen
        //创建DemoNetConnection初始化，并添加到ClientConnections
        //创建FInMemoryReplay（管理所有replay数据），初始化FileAr（用于存Packet）
        FInMemoryNetworkReplayStreamer::StartStreaming
        //生成SpectatorController绑定Connection
        UDemoNetDriver::SpawnDemoRecSpectator

```