# Movement

``` c++

CalculateInput()
    MovementComponent::AddInputVector()

CharacterMovementComponent::TickComponent()
    FVector InputVector = ConsumeInputVector()
    ScaleInputAcceleration()//apply input to acceleration

    //
PerformMovement()
    PhysWalking()
        //准备好comsumeInput得出的加速度
        CalcVelocity()//计算初始平面速度
            //最大速度限制以及无加速度自然减速
            //考虑地面摩擦以及加速度得出水平速度
        MoveAlongFloor()//根据地面sweep的碰撞信息该帧计算移动
            ComputeGroundMovement()
                MoveAlongFloor()
                /* 根据hit.ImpactNormol求出沿着地面方向的移动向量即可
                
                */
                SlideAlongSurface()
                /*原理和在斜坡上走差不多，不过首先要判断走不上去的坡
                和头顶碰到的斜着的天花板，防止卡到地里，把碰撞法线转为水平方向
                再用类似上面的方法计算移动向量*/
        FindFloor()
```
