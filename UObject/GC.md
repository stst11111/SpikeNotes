# 垃圾回收

## 标记流程

``` c++
void PerformReachabilityAnalysis()
{
    //统计序列化参与GC的对象，用于存储用于序列化uobject的array和weak reference列表。
    FGCArrayStruct* ArrayStruct = FGCArrayPool::Get().GetArrayStructFromPool();
    TArray<UObject*>& ObjectsToSerialize = ArrayStruct->ObjectsToSerialize;
    //1.添加UGCObjectReferencer，添加后可用于在非UObject对象上调用AddReferencedObjects方法。
    ObjectsToSerialize.Add(FGCObject::GGCObjectReferencer);
    //2.调用MarkObjectsAsUnreachable，把所有不带KeepFlags和EInternalObjectFlags::GarbageCollectionKeepFlags标记的对象标记为不可达
    (this->*MarkObjectsFunctions[GetGCFunctionIndex(!bForceSingleThreaded, bWithClusters)])(ObjectsToSerialize, KeepFlags);
    //3.分析可达性
    PerformReachabilityAnalysisOnObjects(ArrayStruct, bForceSingleThreaded, bWithClusters);
}
```

MarkObjectsAsUnreachable：
每个线程平均分配NumObjects个uobject，遍历该线程负责的所有obj。

第三步，调用PerformReachabilityAnalysisOnObjects来判断uobject可达性

``` c++
//记录一处引用
struct FGCReferenceInfo
{
    union
    {
        /** Mapping to exactly one uint32 */
        struct
        {
            //返回的嵌套深度
            uint32 ReturnCount: 8;
            //引用的类型，就是EGCRefenceType
            uint32 Type: 4;
            //这个引用对应的属性在类中的地址偏移
            uint32 Offset: 20;
        };
        /** uint32 value of reference info, used for easy conversion to/ from uint32 for token array */
            uint32 Value;
    };
}
class UClass
{
    struct FGCReferenceTokenStream ReferenceTokenStream;
    {
        //用于存储FGCReferenceInfo引用信息
        TArray  Tokens;
    }
    
    void AssembleReferenceTokenStream(bool bForce)
    {
        for( TFieldIterator It(this,); It; ++It)
        {
            UProperty* Property = *It;
            Property->EmitReferenceInfo(*this, 0, EncounteredStructProps);
        }
        if (UClass* SuperClass = GetSuperClass())
        {
            //调用父类
            SuperClass->AssembleReferenceTokenStream();
            //并把父类的stream添加到自己的stream之前
            PrependStreamWithSuperClass(*SuperClass);
        }
        //之后不会再变了Shrink一下
        ReferenceTokenStream.Shrink();
        ClassFlags |= CLASS_TokenStreamAssembled;
    }
}

class TFastReferenceCollector
{
    //可达性分析
    void CollectReferences(FGCArrayStruct& ArrayStruct)
    {
    }
    //遍历uobj的 token stream查找引用。
    void ProcessObjectArray()
    {
    }
}
```
