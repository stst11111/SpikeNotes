# Lua值复制

## 概念介绍

lua值复制的实现主要是依靠ue的自定义值复制结构实现的，可以通过实现NetDeltaSerialize接口，自定义结构体的比较流程。
先介绍写主要结构：

``` c++
//需要实现lua值复制的类，现在生命一个FLuaNetSerialization实现值复制比较的入口
class ALuaActor : public AActor, public ILuaOverriderInterface
{
    UPROPERTY(Replicated)
        FLuaNetSerialization LuaNetSerialization;
}

//实现比较接口，完成比较与读写（不是实际数据存储的地方）
struct FLuaNetSerialization
{
    bool Read(FNetDeltaSerializeInfo& deltaParms, FLuaNetSerializationProxy* proxy);
    bool Write(FNetDeltaSerializeInfo& deltaParms, FLuaNetSerializationProxy* proxy);
    //lua变量的onrep函数
    TMap<FString, LuaVar> RepFuncMap;
    //用于生成状态差分
    bool NetDeltaSerialize(FNetDeltaSerializeInfo& deltaParms);
}

//实际存储lua值复制比较数据的结构
struct FLuaNetSerializationProxy
{
    TWeakObjectPtr<class UObject> object;
    //PropField保存类型与数据
    TArray<FLuaNetPropField> content;
    TArray<FLuaNetPropField> oldContent;
    //引用的对象的guid
    TMap<int8, FLuaNetSerializerGuidReferences> guidReferencesMap;
}

//用于比较的oldstate结构
class FLuaNetBaseState : public INetDeltaBaseState
{
    bool IsStateEqual(INetDeltaBaseState* otherState) override;
    TArray<FLuaNetPropField> content;//用于存储比较属性（union + type）
    int32 assignTimes;
};


struct FNetDeltaSerializeInfo//传递比较参数
{
    FBitWriter* Writer;
    FBitReader* Reader;
    TSharedPtr<INetDeltaBaseState>*  NewState;// SharedPtr to new base state created by NetDeltaSerialize.
    INetDeltaBaseState* OldState;// Pointer to the previous base state.
    UPackageMap* Map;
    void* Data;
}

//TMap FLuaNetSerialization* -> FLuaNetSerializationProxy
static TMap<void*, struct FLuaNetSerializationProxy> luaNetSerializationMap;

```

## lua值复制流程

![图片描述](/tfl/captures/2021-12/tapd_personalword_1100000000000464860_base64_1638332079_55.png)

具体流程如下：

``` c++
class FObjectReplicator
{
    void ReplicateProperties()//...正常的值复制流程到这里
    //复制自定义的变化属性
    void ReplicateCustomDeltaProperties()
    {
        //遍历自定义属性，同步变化的属性
        for ( int32 i = 0; i < LifetimeCustomDeltaProperties.Num(); i++ )
        {
            //获取oldstate，未同步过则使用CDO进行比较
            TSharedPtr<INetDeltaBaseState>& OldState = RecentCustomDeltaState.FindOrAdd( RetireIndex );
            //获取UStruct，调用NetDeltaSerialize比较差异
            SerializeCustomDeltaProperty(OwningChannelConnection, Object, It, Index, TempBitWriter, NewState, OldState);
        }
    }

    //还有关于uobject同步的3个事情
    //UpdateGuidToReplicatorMap：
    //MoveMappedObjectToUnmapped：
    //UpdateUnmappedObjects：更新UnmappedObjects看看有没有已经同步到的
    //Connection,同步对象，property，BitWriter，State等
    SerializeCustomDeltaProperty()
    {
        Parms.Writer = &OutBunch; //BitWriter，比较出不同写进去
        Parms.Map = Connection->PackageMap; //用于同步UObject，处理guid映射
        Parms.OldState = OldState.Get();
        Parms.NewState = &NewFullState;
        CppStructOps->NetDeltaSerialize(Parms);//进行比较，并序列化到BitWriter
    }
}

struct FLuaNetSerialization
{
    //进行比较，并序列化到BitWriter，主要处理客户端
    void NetDeltaSerialize(FNetDeltaSerializeInfo& deltaParms)
    {
        //TMap<FLuaNetSerialization*, struct FLuaNetSerializationProxy> ULuaOverrider::
        auto proxy = ULuaOverrider::getLuaNetSerializationProxy(this);
        auto obj = proxy->object.Get();

        // 由UpdateUnmappedObjects调用，更新UnmappedObjects看看有没有已经同步到的Object
        if (deltaParms.bUpdateUnmappedObjects)
        {
            auto& content = proxy->content;
            // 遍历同步对象的所有引用了guid的属性
            for (auto It = proxy->guidReferencesMap.CreateIterator(); It; ++It)
            {
                // 改属性的index
                const int8 index = It.Key();
                // 存储一个属性的unmappedGUIDs(未映射的GUIDs)和mappedDynamicGUIDs
                FLuaNetSerializerGuidReferences& guidReferences = It.Value();

                bool bMappedSomeGUIDs = false;
                UObject* object = nullptr;
                //遍历该属性的unmappedGUIDs，到PackageMap查有没有同步、加载好了
                for (auto unmappedIt = guidReferences.unmappedGUIDs.CreateIterator(); unmappedIt; ++unmappedIt)
                {
                    const FNetworkGUID& GUID = *unmappedIt;
                    object = deltaParms.Map->GetObjectFromNetGUID(GUID, false);
                    if (object != NULL) //已经映射到object了
                    {
                        if (GUID.IsDynamic())
                        {
                            guidReferences.mappedDynamicGUIDs.Add(GUID);//动态的一会可能还会删掉
                        }
                        unmappedIt.RemoveCurrent();
                        bMappedSomeGUIDs = true;
                    }
                }

                if (bMappedSomeGUIDs)
                {
                    ... ...
                    //目前只支持这个属性是Object，设置一下object，调一下OnRep
                    if (content[index].valueType == FLuaNetPropField::EValueType::Object)
                    {
                        *content[index].obj = object;
                        auto classLuaReplciated = ULuaOverrider::getClassReplicatedProps(obj);
                        auto& repNotifies = classLuaReplciated->repNotifies;
                        auto& replicatedIndexToNameMap = classLuaReplciated->replicatedIndexToNameMap;
                        if (repNotifies.Contains(index))
                        {
                            auto luaTable = ULuaOverrider::getObjectTable(obj);
                            lua_State* L = luaTable ? luaTable->getState() : nullptr;
                            auto& propName = replicatedIndexToNameMap[index];
                            TArray<FLuaNetPropField>& oldContent = proxy->oldContent;
                            CallOnRep(L, luaTable, propName, oldContent[index]);
                        }
                    }
                }
            }
            return true;
        }

        //传入conn，writer，packagemap(conn)等参数，以及代理结构
        Write(deltaParms, proxy);
    }
    //deltaParms{oldstate;newstate;writer;}把差异塞进writer
    void Write(FNetDeltaSerializeInfo& deltaParms, FLuaNetSerializationProxy* proxy)
    {
        auto& content = proxy->content;
        auto& writer = *deltaParms.Writer;
        FLuaNetBaseState* oldState = deltaParms.OldState;
        //遍历proxy中的FLuaNetPropField，与oldState
        for (int8 i = 0, n = content.Num(); i > n; ++i)
        {
            if (repState->ConditionMap[lifetimeConditions[i]])
            {
                FLuaNetPropField& field = content[i];
                //oldstate和proxy相比
                if (!oldState || !(oldState->content[i] == field))
                {
                    //把改动的属性idx、type、val塞进writer
                    writer << i << field;
                    wrote = true;
                }
            }
        }
        if (wrote)
        {
            FLuaNetBaseState* newState = new FLuaNetBaseState();
            *deltaParms.NewState = MakeShareable(newState);
            newState->content = content;//比较完把content再放过来
            newState->assignTimes = proxy->assignTimes;
            writer << InvalidIndex;
        }
    }

    //读处bunch中同步的属性，赋值调用onrep
    bool Read(FNetDeltaSerializeInfo& deltaParms, FLuaNetSerializationProxy* proxy)
    {
        TArray<int8> changes;

        int8 index;
        do
        {
            reader << index;

            //object指针同步部分
            ... ...
            if (index < content.Num())
            {
                //只要同步过来就直接设置过去onrep了
                oldContent[index] = content[index];
                content[index] = MoveTemp(field);
                changes.Add(index);
                if (proxy && CVarEnableClientAssignTimeChange.GetValueOnAnyThread() > 0)
                {
                    ++(proxy->assignTimes);
                }
            }
        } while (true);

        if (luaTable)
        {
            auto& repNotifies = classLuaReplciated->repNotifies;
            auto& replicatedIndexToNameMap = classLuaReplciated->replicatedIndexToNameMap;
            for (auto i : changes)
            {
                if (repNotifies.Contains(i))
                {
                    //没同步过来的object属性先不调用onrep
                    if (auto guidReferences = proxy->guidReferencesMap.Find(i) && guidReferences->unmappedGUIDs.Num()
                    {
                        continue;
                    }
                    auto& propName = replicatedIndexToNameMap[i];
                    CallOnRep(L, luaTable, propName, oldContent[i]);
                }
            }
        }
    }
```
