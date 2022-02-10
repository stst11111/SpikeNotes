# lua RPC

在运行时动态创建新的RPC UFunction，并加到UClass里。

## 主要流程

之前在 __slua重要概念-lua重写函数原理__ 中介绍到的，收集lua覆写蓝图的函数之后，通过addClassRPC添加RPC。

收集、构造rpc UFunction过程：

``` c++
class LuaOverrider{

    //递归获取luaModule
    bool addModuleRPCRecursive(lua_State* L, UClass* cls, const LuaVar& luaModule, const LuaVar& cppSuperModule)
    {
        bool bAdded = false;
        if (!luaModule.isTable() || luaModule == cppSuperModule) { return bAdded; }

        LuaVar luaImpl = getLuaImpl(L, luaModule);
        if (luaImpl.isTable())
        {
            LuaVar luaSuperModule = luaImpl.getFromTable<LuaVar>("__super");
            bAdded |= addModuleRPCRecursive(L, cls, luaSuperModule, cppSuperModule);
        }

        bAdded |= addClassRPCByType(L, cls, luaModule, TEXT("MulticastRPC"), FUNC_NetMulticast);
        bAdded |= addClassRPCByType(L, cls, luaModule, TEXT("ServerRPC"), FUNC_NetServer);
        bAdded |= addClassRPCByType(L, cls, luaModule, TEXT("ClientRPC"), FUNC_NetClient);

        return bAdded;
    }

    bool addClassRPCByType(lua_State* L, UClass* cls, const LuaVar& luaModule, FString repType, EFunctionFlags netFlag)
    {
        LuaVar rpcList = luaImpl.getFromTable<LuaVar>(repType);
        TArray<FString> rpcNames;
        //收集所有的TestActor.MulticastRPC.XXX
        while (rpcList.next(key, value))
        {
            if (!key.isString()) { continue; }
            rpcNames.Add(key.asString());
        }
        //遍历所有TestActor.MulticastRPC.XXX构造UFunction
        for (FString& rpcName: rpcNames)
        {
            LuaVar rpcTable = rpcList.getFromTable<LuaVar>(rpcName);
            bool bReliable = rpcTable.getFromTable<bool>("Reliable");
            //已经在c++定义了同名function就忽略
            UFunction* func = cls->FindFunctionByName(FName(*rpcName), EIncludeSuperFlag::ExcludeSuper);
            if (func){ continue;}
            func = NewObject<UFunction>(cls, *rpcName, RF_Public);
            func->FunctionFlags = FUNC_Public | FUNC_Net | netFlag;
            if (bReliable)
            {
                func->FunctionFlags |= FUNC_NetReliable;
            }
            func->ReturnValueOffset = MAX_uint16;
            func->FirstPropertyToInit = NULL;
            func->Next = cls->Children;
            cls->Children = func;
            LuaVar params = rpcTable.getFromTable<NS_SLUA::LuaVar>("Params");
            if (params.isTable())
            {
                LuaVar index;
                LuaVar paramType;
                UField** propertyStorageLocation = &(func->Children);
                //遍历参数列表构造UFunction
                while (params.next(index, paramType))
                {
                    UProperty* newProperty = nullptr;
                    if (paramType.isInt())//基本类型
                    {
                        EPropertyClass propertyClass = (EPropertyClass)paramType.asInt();
                        newProperty = PropertyProto::createProperty(PropertyProto(propertyClass), func);
                        newProperty->SetPropertyFlags(CPF_HasGetValueTypeHash);
                    } 
                    else if (paramType.isUserdata("UScriptStruct"))
                    {
                        UScriptStruct* scriptStruct = paramType.castTo<UScriptStruct*>();
                        if (scriptStruct)
                        {
                            newProperty = PropertyProto::createProperty(PropertyProto(EPropertyClass::Struct, scriptStruct), func);
                        }
                    }
                    else if (paramType.isUserdata("UClass"))
                    {
                        UClass* propertyClass = paramType.castTo<UClass*>();
                        if (propertyClass)
                        {
                            newProperty = PropertyProto::createProperty(PropertyProto(EPropertyClass::Object, propertyClass), func);
                        }
                    }
                    else if (paramType.isTable())
                    {   //两个叠加类型，一般第一个是数组
                        LuaVar mainType = paramType.getAt(1);
                        LuaVar subType = paramType.getAt(2);
                        if (mainType.isInt())
                        {
                            EPropertyClass propertyClass = (EPropertyClass)mainType.asInt();
                            if (subType.isUserdata("UScriptStruct"))
                            {
                                UScriptStruct* scriptStruct = subType.castTo<UScriptStruct*>();
                                if (scriptStruct)
                                {
                                    newProperty = PropertyProto::createProperty(PropertyProto(propertyClass, scriptStruct), func);
                                }
                            }
                            else if (subType.isUserdata("UClass"))
                            {
                                UClass* subClass = subType.castTo<UClass*>();
                                if (subClass)
                                {
                                    newProperty = PropertyProto::createProperty(PropertyProto(propertyClass, subClass), func);
                                }
                            }
                            else if (subType.isInt())
                            {
                                EPropertyClass typeEnum = subType.castTo<EPropertyClass>();
                                newProperty = PropertyProto::createProperty(PropertyProto(propertyClass, typeEnum), func);
                            }
                        }
                    }

                    if (newProperty)
                    {
                        newProperty->SetPropertyFlags(CPF_Parm);

                        *propertyStorageLocation = newProperty;
                        propertyStorageLocation = &(newProperty->Next);
                    }
                }
            }

            func->StaticLink(true);

            cls->AddFunctionToFunctionMap(func, *rpcName);
            cls->NetFields.Add(func);
#if WITH_EDITOR
            luaRPCFuncs.Add(func);
#endif
            func->SetNativeFunc((Native)&ULuaOverrider::luaOverrideFunc);
            func->Script.Insert(Code, sizeof(Code), 0);

            bAdded = true;
        }
        return bAdded;
    }
}
```

作为构造UFunction的参数结构

``` c++
//作为构造UFunction的参数结构
struct PropertyProto {
    EPropertyClass type;
    EPropertyClass subType;//表示TArray参数的时候作为ArrayProperty的inner
    UClass* cls;//表示UObject的具体类
    UScriptStruct* scriptStruct;
    //创建UProperty
    UProperty* createProperty(const PropertyProto& proto, UObject* outer) {
        UProperty* p = nullptr;
        if (!outer)
        {
            outer = getPropertyOutter();
        }

        switch (proto.type) {
            
        }
        return p;
    }
}

```
