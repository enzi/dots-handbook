### Addendum for System dependencies

It was mentioned that systems complete dependencies in `BeforeOnUpdate` and gather dependencies in `AfterOnUpdate`.
While this is true, it leads to some wrong expectations.

It's easiest to explain with an example.

Imagine 2 systems A and B. Both work on a singleton NativeList and register a write dependency. (For some reason
we don't go into detail, system A can't schedule a job but has to do it on the main thread.)

System A does main thread tasks, gets the singleton, iterates through the list, processes element and then clears the list.
System B updates after system A and schedules a job, and the job adds to the NativeList.

System A
```csharp
protected override void OnCreate()
{
    // stock method to create RW dependency in SystemState, there are other and better methods
    SystemAPI.GetSingletonRW<RequestSingleton>();
}

protected override void OnUpdate()
{
    var requests = SystemAPI.GetSingleton<RequestSingleton>().Requests;

    if (requests.Length == 0)
        return;	        
    ...    
    request.Clear();
}
```

System B
```csharp
public void OnUpdate(ref SystemState state)
{
    var requests = SystemAPI.GetSingleton<RequestSingleton>().Requests;

    new AddToListJob()
    {
        Requests = requests
    }.Schedule();
}
```

This code will throw a safety problem in `requests.Length == 0` that `AddToListJob` is writing to the singleton. But why?

System A has a RW dependency on `RequestSingleton` and we learned `BeforeOnUpdate` calls `.Complete`, so any write dependency
should be completed by the `.Complete` handle. Also, the job has long been finished when System A reads the singleton.
And why does the `.Complete` in System A `BeforeOnUpdate` not pick up the job when it has a clear dependency?

> When System A reads the singleton, the job handle for System B has not yet been completed
because System B `BeforeOnUpdate` hasn't been called yet. This makes the safety system think that a job is still running.

But calling `Dependency.Complete();` in System A fixes the problem!

> The complete dependency call in `BeforeOnUpdate` is NOT the same as calling `Dependency.Complete()` in `OnUpdate`!
We need to look at code from `SystemState.cs` to get a better understanding.

```csharp
public JobHandle Dependency
{
    get
    {
        if (NeedToGetDependencyFromSafetyManager)
        {
            var depMgr = m_DependencyManager;
            NeedToGetDependencyFromSafetyManager = false;
            m_JobHandle = depMgr->GetDependency(m_JobDependencyForReadingSystems.Ptr,
                m_JobDependencyForReadingSystems.Length, m_JobDependencyForWritingSystems.Ptr,
                m_JobDependencyForWritingSystems.Length, clearReadFencesAfterCombining:false);
        }

        return m_JobHandle;
    }
    set
    {
        NeedToGetDependencyFromSafetyManager = false;
        m_JobHandle = value;
    }
}
```
and
```csharp
internal void BeforeOnUpdate()
{
    BeforeUpdateVersioning();

    // We need to wait on all previous frame dependencies, otherwise it is possible that we create infinitely long dependency chains
    // without anyone ever waiting on it
    m_JobHandle.Complete();
    NeedToGetDependencyFromSafetyManager = true;
}
```

Notice `NeedToGetDependencyFromSafetyManager`. During `BeforeOnUpdate` m_JobHandle, which is the same as
`state.Dependency` in ISystem or `Dependency` in SystemBase, is called.

During that, `NeedToGetDependencyFromSafetyManager` is `false`, so it doesn't get an updated dependency as it's not
going through the while `if(NeedToGetDependencyFromSafetyManager)`
block. This means, all handles are completed BUT not handles that were registered after it.

This makes a big difference because calling `Dependency.Complete()` in `OnUpdate` does indeed get an updated
dependency (as `NeedToGetDependencyFromSafetyManager` is set to `true` at the end of `BeforeOnUpdate`) which would have
handles that came after it too.

So to fix this problem there are a few options:
- merge both systems into one, which would solve the problem that system A doesn't know about system B's job
- call `Dependency.Complete();` in SystemA
- or `EntityManager.CompleteDependencyBeforeRW<RequestSingleton>();` in SystemA 