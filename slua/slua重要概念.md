
# slua重要概念

## Lua对象模型

属性访问：

LuaOverrider::__index

通过__cppinst找到userdata，强制转换成UObject。

LuaObject::objectIndex

拿到obj的UClass，在LuaState::classMap里面找。

## Slua GC

UE和lua都有各自的gc，UE和lua又可以互相持有对方的对象。因此需要UE和lua需要知道对方持有了哪些对象来避免gc调对方正在引用的对象。

### ue维护lua的引用

lua持有的ue对象都是通过ue push过来的，此时就会通过addRef添加引用到LuaState::objRefs，当gc的时候FGCObject就会通过AddReferencedObjects把要标记的对象放到Collector，从而防止gc。

``` c++
static int pushGCObject(lua_State* L,T obj,const char* tn,lua_CFunction setupmt,lua_CFunction gc,bool ref) {
    if(getFromCache(L,obj,tn)) return 1;
    lua_pushcclosure(L,gc,0);
    int f = lua_gettop(L);
    int r = pushType(L,obj,tn,setupmt,f);
    lua_remove(L,f); // remove wraped gc function
    if (r) {
      //添加引用到LuaState::objRefs
        addRef(L, obj, lua_touserdata(L, -1), ref);
      //cache到lua中位于LUA_REGISTRYINDEX位置的表，key：obj，val：userdata
        cacheObj(L, obj);
    }
    return r;
}
//LuaState为FGCObject，通过AddReferencedObjects标记对象
void LuaState::AddReferencedObjects(FReferenceCollector & Collector)
{
    for (UObjectRefMap::TIterator it(objRefs); it; ++it)
    {
        UObject* item = it.Key();
        GenericUserData* userData = it.Value();
        if (userData && !(userData->flag & UD_REFERENCE))
            continue;
        Collector.AddReferencedObject(item);
    }
}
```

当lua中没有变量指向这个userdata，lua这边会把userdata给gc掉，会调用元表中绑定的 `LuaObject::gcObject` ，进而调用到`LuaState::unlinkUObject`，移除objRefs上的引用。

``` c++
void LuaState::unlinkUObject(const UObject * Object,void* userdata)
{
    auto udptr = objRefs.Find(Object);
    if (!udptr) return;
    GenericUserData* ud = *udptr;
    if (userdata && userdata != (void*)ud) return;
    //移除引用
    objRefs.Remove(const_cast(Object));
    if (!ud || ud->flag & UD_HADFREE) return;
    ud->flag |= UD_HADFREE;
    ensure(ud->ud == Object);
    LuaObject::removeObjCache(L, (void*)Object);
}
```

## Lua Override流程

![图片描述](/tfl/captures/2021-11/tapd_personalword_1100000000000464860_base64_1635910024_52.png)
参考资料：
UFunction解析：[UE4对象系统_UFunction - 简书 (jianshu.com)](https://www.jianshu.com/p/4cffb2349c22?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)
蓝图解析：[UE4蓝图解析（二） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/69552168)

### lua重写函数原理

Ex_LuaOverride字节码绑定了luaOverrideFunc函数，该函数根据栈帧数据，在绑定的lua table中查找函数，参数入栈执行函数，并在FuncMap中把lua函数以LuaVar的形式缓存起来，至此便是cpp/蓝图侧调用lua重写的函数的流程。
源码分析：

``` c++

class ULuaOverrider：
{
bool bindOverrideFuncs(const UObjectBase* obj, UClass* cls, bool bCDOLua) 
{
    //1.递归Lua父类，收集所有lua定义的函数名到funcNames
    getLuaFunctions(L, funcNames, luaModule);
    //2.如果cpp/蓝图类中有该函数，进行绑定
    for (FString& funcName : funcNames){
        UFunction* func = cls->FindFunctionByName(FName(*funcName), EIncludeSuperFlag::IncludeSuper);
        //FUNC_BlueprintEvent | FUNC_Net
        if (func && (func->FunctionFlags & OverrideFuncFlags)) 
            //把luaOverrideFunc和该类函数绑定
            if (hookBpScript(func, cls, (Native)&ULuaOverrider::luaOverrideFunc)) 
                hookCounter++;
    }
}

//递归父类获取lua函数名到funcNames
void getLuaFunctionsRecursive(lua_State* L, TSet& funcNames)
{
}

//绑定Lua函数（向overrideFunc中添加字节码）
bool LuaOverrider::hookBpScript(UFunction* func, UClass* cls, Native hookFunc)
{
    //注册字节码对应函数
    GRegisterNative(Ex_LuaOverride, hookFunc);
    //用函数名获取子类函数
    UFunction* overrideFunc = cls->FindFunctionByName(func->GetFName(), EIncludeSuperFlag::ExcludeSuper);
    //存在子类函数，向字节码中插入执行lua函数的字节码
    if (overrideFunc == func)
        overrideFunc->Script.Insert(Code, sizeof(Code), 0);
    else if(!overrideFunc)
    {
        //复制创建UFunction对象，添加字节码
        overrideFunc = duplicateUFunction(func, cls, func->GetFName(), (Native)&ULuaOverrider::luaOverrideFunc);
        static TArray ShortCode(Code, sizeof(Code));
        overrideFunc->Script = ShortCode;
    }
}

//用模板函数复制出一个UFunction对象，并且添加到class
UFunction* duplicateUFunction(UFunction* templateFunction, UClass* outerClass, FName newFuncName, Native nativeFunc)
{
}

//执行被绑定的Lua函数（Stack：当前函数栈帧信息）
void LuaOverrider::luaOverrideFunc(FFrame& Stack)
{
    //获取对应的蓝图节点
    UFunction* func = Stack.Node;
    //执行函数的对象
    UObject* obj = Stack.Object;
    //局部变量
    uint8* locals = Stack.Locals;
    if (Stack.CurrentNativeFunction != func)
    {
        obj = this;
        func = Stack.CurrentNativeFunction;
        locals = (uint8*)RESULT_PARAM;
    }
    auto& tableMap = objectTableMap.FindChecked(L);
    NS_SLUA::LuaVar* table = tableMap.Find(obj);
    //直接元表index取函数，在ILuaOverriderInterface::FuncMap中缓存
    LuaVar luaFunc = getLuaFunction(L, obj, table, func->GetName());
    //调用lua函数
    luaFunc.callByUFunction(func, locals, Stack.OutParms, RESULT_PARAM, luaSelfTable);
}

bool LuaVar::callByUFunction(UFunction* func,uint8* parms,FOutParmRec *outParams,RESULT_DECL,LuaVar* pSelf) 
{
    //self入栈
    pSelf->push();n++;
    // 参数入栈
    for(TFieldIterator it(func);it && (it->PropertyFlags&CPF_Parm);++it) {
          UProperty* prop = *it;
          uint64 propflag = prop->GetPropertyFlags();
          //...检查propflag，返回值跳过
        //传入参数UProperty对象以及地址，通过对应pusher入栈
          pushArgByParms(prop,parms+prop->GetOffset_ForInternal());
        n++;
      }
    int remain = retCount = docall(n);
    //把栈内的返回值check到outParams
    for (TFieldIterator it(func); remain > 0 && it && (it->PropertyFlags & CPF_Parm); ++it) {
            UProperty* prop = *it;
            uint64 propflag = prop->GetPropertyFlags();
            if (IsRealOutParam(propflag)) {
                checkOutputValue(prop);
            }
      }
}
```
