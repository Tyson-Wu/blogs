---
title: 摇杆实现方法1
date: 2020-12-23 22:01:00
categories:
- [Unity, Component]
tags:
- JoyStick
---

![wwwwwww](/blogs/images/src/ezgif.com-gif-maker-1.gif)
自己做了个飞行模拟的小游戏，发布到手机上需要用到摇杆来控制，所以自己写了一个Joystick类，代码如下：

``` makedown
using UnityEngine;
using UnityEngine.EventSystems;
using System;

/// <summary>
/// UGUI游戏摇杆
/// </summary>
public class Joystick : MonoBehaviour, IPointerDownHandler, IPointerUpHandler, IDragHandler
{
    public static Joystick instance;
    [SerializeField] Camera uiCamera;//摇杆对应的ui摄像机
    [SerializeField] RectTransform _triggerArea;//触发启动摇杆区域
    [SerializeField] RectTransform _joyStickArea;//摇杆活动区域
    [SerializeField] RectTransform centerTrans;//摇杆图标
    [SerializeField] float sensitivity = 1;//摇杆灵敏度
    [SerializeField] float triggerAreaScale = 1;//摇杆控制区域
    /// <summary>
    /// 摇杆偏移方向
    /// </summary>
    public Vector2 moveDir { get; private set; }
    /// <summary>
    /// 摇杆偏移长度[0-1]
    /// </summary>
    public float moveLen { get; private set; }
    /// <summary>
    /// 是否在操控摇杆
    /// </summary>
    public bool isDraging { get; private set; }
    /// <summary>
    /// 摇杆横向偏移量
    /// </summary>
    public float Horizontal { get { return moveDir.x * moveLen; } }
    /// <summary>
    /// 摇杆纵向偏移量
    /// </summary>
    public float Vertical { get { return moveDir.y * moveLen; } }

    Canvas _canvas;
    Canvas canvas { get { if (_canvas == null) _canvas = transform.GetComponentInParent<Canvas>(); return _canvas; } }
    Vector3 targetScreenPointPos = Vector3.zero;
    

    float radius = 0f;
    Vector3 centerScreenPos;
    float depth { get { return centerScreenPos.z; } }
    bool hasInit = false;

    public Action onJoyStickPointerUp;
    private void Start()
    {
        instance = this;

    }
    void OnEnable()
    {
        InitParam();
    }
    void OnDisable()
    {
        isDraging = false;
        targetScreenPointPos = centerScreenPos;
        centerTrans.localPosition = Vector3.zero;
    }
    void InitParam()
    {
        if (hasInit) return;
        hasInit = true;
        centerScreenPos = uiCamera.WorldToScreenPoint(_joyStickArea.position);
        Rect rect = RectTransformUtility.PixelAdjustRect(_joyStickArea, canvas);
        radius = triggerAreaScale * rect.width/2;
        targetScreenPointPos = centerScreenPos;
    }
    public void OnPointerDown(PointerEventData eventData)
    {
        InitParam();
        Vector2 pointScreenPos = eventData.position;
        //if (Vector2.Distance(pointScreenPos, centerScreenPos) > radius) return;

        isDraging = true;
        
        
        //targetScreenPointPos = uiCamera.WorldToScreenPoint(eventData.pointerCurrentRaycast.worldPosition);

    }
    public void OnDrag(PointerEventData eventData)
    {
        if (!isDraging) return;
        Vector3 pointScreenPos = eventData.position;
        pointScreenPos.z = depth;
        Vector3 dir = pointScreenPos - centerScreenPos;
        float len = dir.magnitude;
        len = len > radius ? radius : len;
        dir = len * dir.normalized;
        targetScreenPointPos = centerScreenPos + dir;
    }

    public void OnPointerUp(PointerEventData eventData)
    {
        isDraging = false;
        targetScreenPointPos = centerScreenPos;
        onJoyStickPointerUp?.Invoke();
    }
    
    private void Update()
    {
        Vector3 screenPointPos = uiCamera.WorldToScreenPoint(centerTrans.position);
        if(Vector3.Distance(screenPointPos, targetScreenPointPos)<4)
        {
            screenPointPos = targetScreenPointPos;
        }
        else
        {
            screenPointPos = Vector3.Lerp(screenPointPos, targetScreenPointPos, sensitivity * Time.deltaTime);
        }

        centerTrans.position = uiCamera.ScreenToWorldPoint(screenPointPos);
        moveDir = screenPointPos - centerScreenPos;
        moveDir /= radius;
        moveLen = moveDir.magnitude;
        moveLen = moveLen > 1 ? 1 : moveLen;
    }
}
```

通过属性器可以获取当前摇杆的状态参数，包括偏移方向以及强度，方向是360度，强度是[0-1]。还有几个`SerializeField`面板设置参数，结构如下图：
![](/blogs/images/src/微信图片_20201223220715.png)