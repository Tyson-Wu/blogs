---
title: 摇杆实现方法2
date: 2020-12-23 22:02:00
categories:
- [Unity, Component]
tags:
- JoyStick
---

自己做了个飞行模拟的小游戏，发布到手机上需要用到摇杆来控制。原先是通过一个Joystick类来实现，摇杆位置固定，详细请参考[这里](https://tyson-wu.github.io/blogs/2020/12/23/Joy-Stick-Function/)
但是后面策划说要做成王者荣耀那种，摇杆的位置会根据手指的位置有一定的活动区域，所以我重新写了一份逻辑，并且对代码进行一定的逻辑细分，效果如下：
![wwwwwww](/blogs/images/src/ezgif.com-gif-maker-2.gif)

首先，我这里将摇杆分为几个部分：

- 摇杆可触发区域，紫色区域，当手指放在该区域时，可以操控摇杆
- 摇杆中心位置可活动区域，黄色区域，绿色的圆形摇杆背景图的中心点可到达的区域
- 最后是白色的摇杆和绿色摇杆背景的组合

前两个都挂载了一个区域判断的类-`TriggerAreaHandler`，主要用来判断屏幕点是否在该区域内，如图所示：
![wwwwwww](/blogs/images/src/微信图片_20201223224837.png)
详细代码如下：
``` c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
/// <summary>
/// 用于UI区域判定
/// </summary>
public class TriggerAreaHandler : MonoBehaviour
{
    RectTransform[] _triggerArea = null;//触发区域
    Camera _camera = null;//画布摄像机
    Canvas _canvas = null;//画布
    void InitTriggerArea()
    {//初始化区域
        if (_triggerArea != null) return;
        _triggerArea = GetComponentsInChildren<RectTransform>();
        _canvas = GetComponentInParent<Canvas>();
        _camera = _canvas.worldCamera;
    }
    /// <summary>
    /// 计算屏幕区域是否在当前区域内
    /// </summary>
    /// <param name="screenPos">屏幕坐标</param>
    /// <returns></returns>
    public bool CheckScreenPosInArea(Vector2 screenPos)
    {
        InitTriggerArea();
        for (int i=0;i< _triggerArea.Length;++i)
        {
            if (RectTransformUtility.RectangleContainsScreenPoint(_triggerArea[i], screenPos, _camera))
                return true;
        }
        return false;
    }
    /// <summary>
    /// 将屏幕点夹紧到该区域
    /// </summary>
    /// <param name="screenPos"></param>
    /// <returns></returns>
    public Vector2 Clamp(Vector2 screenPos)
    {
        InitTriggerArea();
        Vector2 clampPos = _camera.WorldToScreenPoint(_triggerArea[0].position);
        float len = float.MaxValue;
        InitTriggerArea();
        for (int i = 0; i < _triggerArea.Length; ++i)
        {
            RectTransform rectTransform = _triggerArea[i];
            if (RectTransformUtility.RectangleContainsScreenPoint(_triggerArea[i], screenPos, _camera))
            {
                return screenPos;
            }
            else
            {

                Vector2 rectPos = _camera.WorldToScreenPoint(rectTransform.position);
                Rect rect = RectTransformUtility.PixelAdjustRect(rectTransform, _canvas);
                Vector2 clampPos_temp = Clamp(screenPos, rectPos, new Vector2(rect.width/2, rect.height/2));
                float distance = (clampPos_temp - screenPos).sqrMagnitude;
                if (distance < len)
                {
                    len = distance;
                    clampPos = clampPos_temp;
                }
            }
        }
        return clampPos;
    }
    Vector2 Clamp(Vector2 value,Vector2 center, Vector2 halfSize)
    {
        return new Vector2(Clamp(value.x, center.x - halfSize.x, center.x + halfSize.x),
            Clamp(value.y, center.y - halfSize.y, center.y + halfSize.y));
    }
    float Clamp(float value, float min, float max)
    {
        value = value > min ? value : min;
        value = value < max ? value : max;
        return value;
    }
}
```
第三部分的摇杆组合，主要是用来显示的，所以下面挂载了一个控制其显示的脚本-`JoyStickView`，该脚本控制摇杆组合的位置显示，如图所示：
![wwwwwww](/blogs/images/src/微信图片_20201223225034.png)
代码如下：
``` c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
/// <summary>
/// 控制摇杆显示
/// </summary>
public class JoyStickView : MonoBehaviour
{
    [SerializeField] RectTransform _joyStickCenter;//摇杆的端点
    [SerializeField] RectTransform _joyStickOutline;//摇杆端点区域背景,设置为摇杆端点的父节点
    [SerializeField] float _joyStickAreaScale = 1;//在背景区域的基础上进行缩放，得到一个活动区域

    bool _initSize = false;
    Canvas _canvas = null;
    Vector3 _joyStickOriginScreenPos0 = Vector3.zero; //摇杆初始原点
    Vector3 _joyStickOriginScreenPos = Vector3.zero; //摇杆原点
    float _joyStickRadius = 0;//摇杆真实的屏幕活动半径
    /// <summary>
    /// 获取摇杆屏幕活动半径
    /// </summary>
    /// <returns></returns>
    public float GetJoyStickRadius()
    {
        return _joyStickRadius;
    }
    /// <summary>
    /// 计算背景区域的屏幕尺寸
    /// 在所有函数之前调用
    /// </summary>
    public void CalculateAreaSize()
    {
        if (_initSize) return; 
        _initSize = true;
        if(_canvas==null)  _canvas = GetComponentInParent<Canvas>();
        _joyStickOriginScreenPos0 = _canvas.worldCamera.WorldToScreenPoint(_joyStickOutline.position);
        _joyStickOriginScreenPos = _joyStickOriginScreenPos0;
        SetJoyStickOriginPos(_joyStickOriginScreenPos0);
        Rect rect = RectTransformUtility.PixelAdjustRect(_joyStickOutline, _canvas);
        _joyStickRadius = _joyStickAreaScale * rect.width / 2;
    }
    /// <summary>
    /// 重置摇杆原点为初始原点
    /// </summary>
    /// <returns>摇杆中心点在屏幕上的位置</returns>
    public Vector2 ResetJoyStickOrigionPos()
    {
        SetJoyStickOriginPos(_joyStickOriginScreenPos0);
        return _joyStickOriginScreenPos0;
    }
    /// <summary>
    /// 设置整个摇杆的位置
    /// </summary>
    /// <param name="screenPos">屏幕坐标</param>
    public void SetJoyStickOriginPos(Vector3 screenPos)
    {
        _joyStickOriginScreenPos.x = screenPos.x;
        _joyStickOriginScreenPos.y = screenPos.y;
        _joyStickOutline.position = _canvas.worldCamera.ScreenToWorldPoint(_joyStickOriginScreenPos);
    }
    /// <summary>
    /// 基于偏移向量，设置摇杆显示位置
    /// </summary>
    /// <param name="dir">移动向量</param>
    public void SetJoyStick(Vector2 dir)
    {
        Vector3 dir3 = dir.sqrMagnitude > 1 ? dir.normalized : dir;
        Vector3 joyStickScreenPos = dir3 * _joyStickRadius + _joyStickOriginScreenPos;
        _joyStickCenter.position = _canvas.worldCamera.ScreenToWorldPoint(joyStickScreenPos);
    }
    /// <summary>
    /// 判断屏幕坐标是否在摇杆触发区域里
    /// </summary>
    /// <param name="screenPos">屏幕坐标</param>
    /// <returns></returns>
    public bool InJoyStickArea(Vector2 screenPos)
    {
        float len = Vector2.Distance(screenPos, _joyStickOriginScreenPos);
        return  len < _joyStickRadius;
    }
}
```
最后是整个摇杆逻辑实现部分-`JoyStickTrigger`，这个类里面记录了触发事件，以及对外的参数，如图所示：
![](/blogs/images/src/微信图片_20201223223000.png)
同样的，通过属性器可以获取当前摇杆的状态参数，包括偏移方向以及强度，方向是360度，强度是[0-1]。详细代码如下：
``` c#
using UnityEngine;
using UnityEngine.EventSystems;
/// <summary>
/// 摇杆控制模块
/// </summary>
public class JoyStickTrigger : MonoBehaviour, IPointerDownHandler, IPointerUpHandler, IDragHandler
{
    public static JoyStickTrigger instance;
    [SerializeField] TriggerAreaHandler _joyStickCenterArea;//摇杆背景中心点活动区域
    [SerializeField] TriggerAreaHandler _joyStickTriggerArea;//可激活摇杆的区域
    [SerializeField] JoyStickView _joyStick;//摇杆
    [SerializeField] float _sensitivity = 1;//摇杆灵敏度
    [SerializeField] float _triggerAreaScale = 1;//摇杆控制区域
    /// <summary>
    /// 摇杆偏移方向
    /// </summary>
    public Vector2 MoveDir { get; private set; }
    /// <summary>
    /// 摇杆偏移长度[0-1]
    /// </summary>
    public float MoveLen { get; private set; }
    /// <summary>
    /// 是否在操控摇杆
    /// </summary>
    public bool IsDraging { get { return _onJoyStickActive; } }
    /// <summary>
    /// 摇杆横向偏移量
    /// </summary>
    public float Horizontal { get { return MoveDir.x * MoveLen; } }
    /// <summary>
    /// 摇杆纵向偏移量
    /// </summary>
    public float Vertical { get { return MoveDir.y * MoveLen; } }

    Vector2 _originScreenPos = Vector2.zero;//激活摇杆的手指点击位置
    Vector2 _dragScreenPos = Vector2.zero;//实时拖动手指位置
    Vector2 _interpolateScreenPos = Vector2.zero;//实时摇杆位置
    bool _onJoyStickActive = false;//摇杆是否激活
    void Awake()
    {
        instance = this;
    }
    void OnEnable()
    {
        _joyStick.CalculateAreaSize();
    }
    void OnDisable()
    {
        OnPointerUp(null);
    }

    public void OnPointerDown(PointerEventData eventData)
    {
        if (!_joyStickTriggerArea.CheckScreenPosInArea(eventData.position))
            return;
        _onJoyStickActive = true;
        //_originScreenPos = eventData.position;
        _originScreenPos = _joyStickCenterArea.Clamp(eventData.position);
        _interpolateScreenPos = _dragScreenPos = eventData.position;
        _joyStick.SetJoyStickOriginPos(_originScreenPos);
    }
    public void OnDrag(PointerEventData eventData)
    {
        if (!_onJoyStickActive) return;
        _dragScreenPos = eventData.position;
    }

    public void OnPointerUp(PointerEventData eventData)
    {
        _onJoyStickActive = false;
        _dragScreenPos = _joyStick.ResetJoyStickOrigionPos();//手指位置归于原点
        //_interpolateScreenPos += _dragScreenPos - _originScreenPos;//将插值点随摇杆位置一起归位
        _interpolateScreenPos = _dragScreenPos;//
        _originScreenPos = _dragScreenPos;//
    }

    private void Update()
    {
        _interpolateScreenPos = Vector2.Lerp(_interpolateScreenPos, _dragScreenPos, Time.deltaTime * _sensitivity);
        Vector2 dir = _interpolateScreenPos - _originScreenPos;
        dir /= _joyStick.GetJoyStickRadius();
        dir = dir.sqrMagnitude > 1 ? dir.normalized : dir;
        MoveDir = dir;
        MoveLen = dir.magnitude;
        _joyStick.SetJoyStick(dir);
    }
}
```