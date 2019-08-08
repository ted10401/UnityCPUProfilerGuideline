# Unity CPU Usage Profiler 指南

## Player Loop System 簡介

Player Loop System 是在 Unity 2018.1 開放的實驗性新功能，能夠讓開發者修改 Unity 引擎的遊戲循環。

能透過 PlayerLoop.GetDefaultPlayerLoop() 來取得預設系統列表，並使用 PlayerLoop.subSystemList 取得該系統下的對應子系統列表。

預設系統列表包含
* Initialization
* EarlyUpdate
* FixedUpdate
* PreUpdate
* Update
* PreLateUpdate
* PostLateUpate

## Profiler Window

大略講解完 PlayerLoop ，下一步就是打開 Profiler 視窗，並進行初步的理解。  
Profiler 視窗包含了幾個區塊，Profiler Controls、Profiler Timeline、Profiler Data。

### Profiler Controls (Toolbar)

Add Profiler：新增指定的 Profiler 類別  
Record：紀錄資料  
Deep Profile：深度分析  
Profile Editor：啟用 Editor Profiler，針對 Editor 分析  
Editor：選擇分析對象  
Allocation Callstacks：  
Clear on Play：運行時清除資料  
Clear：清除資料  
Load：讀取資料  
Save：儲存資料  
Frame：目前幀  
◀：前一幀  
▶：後一幀  
Current：跳往當前幀  

### Profiler Timeline (The upper part)

目前提供 CPU Usage、GPU、Rendering、Memory、Audio、Video、Physics、Physics2D、NetworkMessages、NetworkOperations、UI、UIDetails、GlobalIllumination 類別

使用 Add Profiler 添加類別後就會在 Profiler Timeline 中顯示該類別的數據圖表。  
並且可以在圖表內拖動目前幀來顯示指定幀的該類別數據資料。

### Profiler Data (The lower part)

根據不同的 Profiler Timeline 可以選擇的顯示模式各有不同。  
此文章針對 CPU Usage Profiler 進行說明。

Hierarchy：選擇顯示模式  
Overview：方法名稱  
Total：表示該方法及子方法的時間總花費佔比  
Self：表示該方法的時間花費佔比  
Calls：表示該方法在指定幀的呼叫次數  
GC Alloc：表示指定幀的被指派記憶體大小，會由 Garbage Collector 進行回收並在累積一定程度後進行釋放，須盡量將此數值保持在 0 B，以避免 Garbage Collector 造成的幀數高峰  
Time ms：表示該方法及子方法的時間總花費  
Self ms：表示該方法的時間花費

## 常見 CPU Usage Overviews

* WaitForTargetFPS：當 CPU 開銷低於目標幀率，會產生此參數維持目標幀率
* Gfx.WaitForPresent：開啟多線程渲染時，GPU 負載過高
* Graphics.PresentAndSync：未開啟多線程渲染時，GPU 負載過高

* GameObject.SetActive() 開關物件時調用
    * GameObject.Deactivate 關閉物件時調用
        * GameObject.ActivateAwakeRecursively 負責遍歷物件
            * MonoBehaviour.OnDisable() 從 OnDisable 調用，負責 MonoBehaviour.OnDisable 事件
    * GameObject.Activate 開啟物件時調用
        * GameObject.ActivateAwakeRecursively 負責遍歷物件
            * MonoBehaviour.OnEnable() 從 OnEnable 調用，負責 MonoBehaviour.OnEnable 事件

* Destroy 刪除物件時調用
    * GameObject.Deactivate 關閉物件時調用
    * MonoBehaviour.OnDestroy() 從 OnDestroy 調用，負責 MonoBehaviour.OnDestroy 事件

* PlayerLoop
    * Camera.Render
        * Culling
            * PostProcessLayer.OnPreCull()
                * PostProcessLayer.OnPreCull()
                    * PostProcessLayer.BuildCommandBuffers() 從 PostProcessLayer 調用，負責 PostProcessing 渲染
            * SceneCulling
                * CullAllVisibleLights
                * CullSceneEvents
        * Drawing
            * Render.OpaqueGeometry
            * Render.TransparentGeometry
            * Render.MotionVectors
            * CommandBuffer.BeforeImageEffectsOpaque
        * CommandBuffer.BeforeImageEffects
            * Post-processing
        * CullResults.CreateSharedRendererScene
        * FinalizeUpdateRendererBoundingVolumes
            * SkinnedMeshFinalizeUpdate
    * EarlyUpdate.UpdatePreloading
        * Loading.UpdatePreloading：場景切換或主動動態加載資源時調用，加載資源越多、越複雜，效能開銷即越大
            * UpdatePreloading
                * Application.Integrate Assets in Background
                    * Preload Single Step
                        * Application.LoadLevelAsync Integrate
                        * Loading.ReadObject：加載場景和物件時調用，包括貼圖、網格、材質、著色器、動畫等
                        * SceneManager.Internal_SceneLoaded
    * FixedUpdate.PhysicsFixedUpdate：Unity 物理模組更新
        * Physics.Simulate：從 FixedUpdate 調用，透過指示物理引擎（PhysX）更新物理當前狀態來執行物理模擬
        * Physics.Processing：從 FixedUpdate 調用，負責所有非布料物理
        * Physics.ProcessingCloth：從 FixedUpdate 調用，負責所有布料物理
        * Physics.FetchResults：從 FixedUpdate 調用，負責從物理引擎收集模擬結果
        * Physics.UpdateBodies：從 FixedUpdate 調用，更新物理對象的位置及旋轉，並送出資訊
        * Physics.ProcessReports：從 FixedUpdate 調用，在 FixedUpdate 結束時運行，不同階段的模擬結果都在此運行，包含 Contacts、Joint Breaks、Trigger，區分為下列四個子階段
            * Physics.TriggerEnterExits：從 FixedUpdate 調用，負責 MonoBehaviour.OnTriggerEnter、MonoBehaviour.OnTriggerExit 事件
            * Physics.TriggerStays：從 FixedUpdate 調用，負責 MonoBehaviour.OnTriggerStay 事件
            * Physics.Contacts：從 FixedUpdate 調用，負責 MonoBehaviour.OnCollisionEnter、MonoBehaviour.OnCollisionExit、MonoBehaviour.OnCollisionStay 事件
            * Physics.JointBreaks：從 FixedUpdate 調用，處理與被破壞關節有關的更新及傳遞
        * Physics.UpdateCloth：從 Update 調用，處理與布料及 Skinned Mesh 有關的更新
        * Physics.Interpolation：從 Update 調用，處理所有物理對象的位置及旋轉插值計算
    * PreUpdate.AIUpdate：更新 NavMesh 模組
        * NavMeshManager
        * NavMesh.Internal_CallOnNavMeshPreUpdate()
    * Update.ScriptRunBehaviourUpdate
        * BehaviourUpdate
            * MonoBehaviour.Update()：從 Update 調用，負責 MonoBehaviour.Update 事件
    * Update.ScriptRunDelayedDynamicFrameRate：使用 Coroutine 時調用
        * CoroutinesDelayedCalls
    * PreLateUpdate.ScriptRunBehaviourLateUpdate
        * LateBehaviourUpdate
            * MonoBehaviour.LateUpdate()：從 LateUpdate 調用，負責 MonoBehaviour.LateUpdate 事件
    * PreLateUpdate.DirectorUpdateAnimationBegin
        * Director.ProcessFrame
            * Animators.Update：Unity Animator 模組更新
                * Animators.ApplyOnAnimatorMove：從 OnAnimatorMove 調用，負責 MonoBehaviour.OnAnimatorMove 事件
                * Animators.ProcessGraphJob：從 Animator 視窗調用，負責處理 Mecanim 可視化編輯相關工作
                * Animators.FireAnimationEventAndBehaviours：從 Animators 調用，負責發送 Animation Event
                * Animators.PrepareFirstPass：Unity 動畫模組從 AnimationClip 讀取開始到將數據放入 MeshSkinning 結束，此為第一個 Pass
        * Director.PrepareFrame
    * PreLateUpdate.DirectorUpdateAnimationEnd
        * Director.ProcessFrame
            * Animators.Update：Unity Animator 模組更新
    * PostLateUpdate.PlayerUpdateCanvases：更新 Canvas 佔用的時間
        * UIEvents.WillRenderCanvases
            * UGUI.Rendering.UpdateBatches
                * Canvas.SendWillRenderCanvases()：當 Canvas 內的元件 SetDirty 時調用
                * TransformChangeDispatch：當 Canvas 內的元件 Transform 改變時調用
    * PostLateUpdate.UpdateAllSkinnnedMeshes：Unity Skinned Mesh 模組更新
    * PostLateUpdate.UpdateAllRenderers：Unity Renderer 模組更新

<h2>資料來源</h2>

Unity - Manual: CPU Usage Profiler
https://docs.unity3d.com/Manual/ProfilerCPU.html

【Unity】Unity 2018のPlayerLoopで、Unityが毎フレーム呼ぶ処理を無効にしたり、Update"前"に独自の処理を追加したり
http://tsubakit1.hateblo.jp/entry/2018/04/17/233000

tsubaki/イベント一覧
https://gist.github.com/tsubaki/5887a9375d99c94fc951f6782b970dff
