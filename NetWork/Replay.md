# 回放

## 录制stream

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

    //遍历所有Replicated的Actor
    for (const TSharedPtr<FNetworkObjectInfo>& ObjectInfo : ActiveObjectSet)
    {
        FNetworkObjectInfo* ActorInfo = ObjectInfo.Get();
        //是否到达录制时间
        if (GetDemoCurrentTime() > ActorInfo->NextUpdateTime)
    }
}
```

DemoNetDriver会每帧检查是否需要保存TickCheckpoint

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
    //处在保存checkpoint中的某个阶段，继续分帧处理快照的录制
    if (ReplayHelper.GetCheckpointSaveState() != FReplayHelper::ECheckpointSaveState::Idle)
    {
        ReplayHelper.TickCheckpoint(ClientConnections[0]);
    }
    else
    {
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
            //复制所有PendingCheckpointActors中的actor
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
        //2.处理被删掉的对象
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
        }
        //4.序列化上一步缓存的Actor的GUID(FReplayHelper::SerializeGuidCache)
        case ECheckpointSaveState::SerializeGuidCache:
        {
        }
    }

    *CheckpointArchive << CurrentLevelIndex;
    *CheckpointArchive << DeletedNetStartupActors;
    SerializeGuidCache( GuidCache, CheckpointArchive );
    WriteDemoFrameFromQueuedDemoPackets( *CheckpointArchive, ClientConnections[0]->QueuedCheckpointPackets);
}

void UDemoNetDriver::WriteDemoFrameFromQueuedDemoPackets( FArchive& Ar, TArray<FQueuedDemoPacket>& QueuedPackets)
{
    //序列化所有levels
    for (StreamingLevel : NewStreamingLevelsThisFrame)
    {
        Ar << PackageName;
        Ar << PackageNameToLoad;
        Ar << StreamingLevel->LevelTransform;
    }
    SaveExternalData( Ar );
    //序列化QueuedPackets
    for ( int32 i = 0; i < QueuedPackets.Num(); i++ )
    {
        WritePacket( Ar, QueuedPackets[i].Data.GetData(), QueuedPackets[i].Data.Num() );
    }
}

void UDemoNetDriver::TickDemoRecord( float DeltaSeconds )
{

}

void UDemoNetConnection::LowLevelSend(void* Data, int32 CountBytes, int32 CountBits)
{
    
}
```
