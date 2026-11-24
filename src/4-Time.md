# 4 Time

## 4.1 Timer

一个`Timer`是一个bevy_time子crate中的类型，代表了bevy中的定时器，通过该类型，我们可以实现计时功能，其主要的两个构造方式如下：

```rust
pub fn new (duration: Duration , mode: TimerMode ) -> Self
pub fn from_seconds (duration: f32 , mode: TimerMode ) -> Self

//其中TimerMode的定义如下
pub enum TimerMode {
    Once,
    Repeating,
}
```

`Timer`并不关心时间的起点，而更关心时间差，其`TimerMode`指定了`Timer`的模式，这个模式的含义如下：

- Once：在经过`Duration`时间后，计时器将停止跟踪并保持在完成状态，直到重置为止。
- Repeating：在经过`Duration`时间后，计时器不会保持状态，而是可以继续计时，再经过`Duration`时间后还会触发，并且仍然可以在任何给定点重置。

`Timer`的用法非常简单，读者可以查看[文档](https://docs.rs/bevy_time/0.17.3/bevy_time/struct.Timer.html#method.from_seconds)，这里只介绍以下几个方法。

```rust
//将计时器的时间往前拨动
//如果拨动的时间大于Timer内的Duration，Once类型的计时器会卡最大的Duration，Repeating类型的计时器不会被影响
pub fn tick (&mut self, delta: Duration ) -> &Self

//计时器从创建，到上一次tick为止，是否已经到达持续时间
pub fn is_finished (&self) -> bool

//仅在上次调用tick方法后，计时器到达Duration的情况下，才返回true
pub fn just_finished (&self) -> bool
```

`is_finished`与`just_finished`的区别可能有些微妙，这可以通过以下代码来演示。一般而言我们会在`Repeating`模式下使用`just_finished`，在`Once`模式下使用`is_finished`。

```rust
//Repeating模式下使用just_finished
let mut timer = Timer::from_seconds(1.0, TimerMode::Repeating);
//我们在这里拨动了1.1s的时间，大于1.0秒，因此在这个tick里满足了条件
timer.tick(Duration::from_secs_f32(1.1));
assert_eq!(timer.just_finished(), true);
//我们又在一个新的tick里拨动了0.5s，现在一共是1.6s，还没达到第二个时间点(2s)
//所以下面会返回false
timer.tick(Duration::from_secs_f32(0.5));
assert_eq!(timer.just_finished(), false);

//Once模式下使用is_finished
//在第一次就已经拨到了1.1s,大于1.0s，因此该始时钟停留在1.0s
//所以is_finished的返回值一直都是true
let mut timer = Timer::from_seconds(1.0, TimerMode::Once);
timer.tick(Duration::from_secs_f32(1.1));
assert_eq!(timer.is_finished(), true);
timer.tick(Duration::from_secs_f32(0.5));
assert_eq!(timer.is_finished(), true);
```

## 4.2  Time

Bevy中内置了一些时间信息，其定义如下，这些`Time`**以全局资源的形式存在**，通过这些我们可以获得Bevy的时间信息，从而指定时间相关的任务，例如定时运行某些系统等。

```rust
pub struct Time<T: Default = ()> {
    context: T,
    wrap_period: Duration,
    delta: Duration,
    delta_secs: f32,
    delta_secs_f64: f64,
    elapsed: Duration,
    elapsed_secs: f32,
    elapsed_secs_f64: f64,
    elapsed_wrapped: Duration,
    elapsed_secs_wrapped: f32,
    elapsed_secs_wrapped_f64: f64,
}
```

Bevy中存在四种`Time`，这些时钟是我们在调用`DefaultPlugins`时内部的 [`TimePlugin`](https://docs.rs/bevy_time/0.17.3/bevy_time/struct.TimePlugin.html)为我们插入的，并且还为我们创建好了更新这些信息的，他们分别是：

- Time<Real>：记录实际经过的时间
- Time<Virtual>：记录虚拟游戏时间，该时间可以暂停或调整
- Time<Fixed>：根据虚拟时间跟踪固定时间步长
- Time：一个通用时钟，对应于系统的“当前”或“默认”时间

​	Bevy是如何做到这些的？简而言之，`TimePlugin`插件在`First`调度中（还记得`First`吗？那是游戏循环的第一个阶段）更新了`Time<Real>`（使用渲染app传递的时间或者直接调用`Instant::now()`），然后使用了这个时间来更新了`Time`和`Time<Virtual>`。

bevy_time的源码里是这么写的，利用`Time<Real>`两次更新的时间差来更新`Time<Virtual>`，然后直接拷贝了一份给`Time`。这说明其实`Time`和`Time<Virtual>`里的时间其实是一样的（除非在`FixedMain`调度里）。

```rust
pub fn update_virtual_time(current: &mut Time, virt: &mut Time<Virtual>, real: &Time<Real>) {
    let raw_delta = real.delta();
    virt.advance_with_raw_delta(raw_delta);
    *current = virt.as_generic();
}
```

在`FixedMain`调度里，Bevy会更改Time，这是通过调用下面这个特殊的系统来实现的。这个系统利用`Time<Virtual>`更新了`Time<Fixed>`，然后在`FixedMain`阶段里直接修改了`Time`的时间与`Time<Fixed>`相同，来让我们在运行`FixedMain`中的系统时，调用`Time`看到的时间是`Time<Fixed>`。最后，当离开这个调度后，我们又将其更正为`Time<Virtual>`，一切就这样恢复原样，剩下的系统看到的`Time`不会发生任何改变。

```rust
pub fn run_fixed_main_schedule(world: &mut World) {
    let delta = world.resource::<Time<Virtual>>().delta();
    world.resource_mut::<Time<Fixed>>().accumulate(delta);

    // Run the schedule until we run out of accumulated time
    let _ = world.try_schedule_scope(FixedMain, |world, schedule| {
        while world.resource_mut::<Time<Fixed>>().expend() {
            *world.resource_mut::<Time>() = world.resource::<Time<Fixed>>().as_generic();
            schedule.run(world);
        }
    });

    *world.resource_mut::<Time>() = world.resource::<Time<Virtual>>().as_generic();
}
```

说了这么多，其实只告诉了我们下面三件事：

**在`RunFixedMainLoop`阶段执行的系统，我们看到的`Time`是`Time<Fixed>`。**

**在`Update`阶段执行的系统，我们看到的`Time`是`Time<Virtual>`。**

**如果需要获得现实世界的时间，我们则需要使用`Time<Real>`。**

## 4.3 Time与Timer的配合使用

### 4.3.1 定时执行系统

很多时候，我们想要创建一个定时任务，利用Timer和Time，我们可以轻轻松松完成这件事，例如下面这样。

```rust
//返回一个闭包，这个闭包会在每次游戏循环里被执行，当返回true时系统将会执行
pub fn on_real_timer(duration: Duration) -> impl FnMut(Res<Time<Real>>) -> bool + Clone {
  	//创建一个定时器
    let mut timer = Timer::new(duration, TimerMode::Repeating);
  	//闭包获取Time<Real>，并拨动始终，判断是否经过了一段时间
    move |time: Res<Time<Real>>| {
        timer.tick(time.delta());
        timer.just_finished()
    }
}

//利用run_if，可以使用这个方法
app.add_system(Update,some_system.run_if(on_real_timer(Durarion::from_sec_32(1.0))))
```

我们不需要重复编写这些功能，Bevy在bevy_time中已经为我们提供了一套常用的conditions，读者可以查看[文档](https://docs.rs/bevy_time/0.17.3/bevy_time/common_conditions/index.html)。

### 4.3.2 定时执行系统（进阶）

上面的方式只适用于简单的情况，更多时候我们还需要进行一定的控制，这时我们也可以使用上面的方式，但是我们这时候需要利用一个`Component`来存储`Timer`并手动利用`tick`更新时间，就像下面这样。

```rust
#[derive(Component, Deref, DerefMut)]
struct AnimationTimer(Timer);

fn animate_sprite(
    time: Res<Time>,
    mut query: Query<&mut AnimationTimer>,
) {
    for mut timer in &mut query {
        timer.tick(time.delta());
        if timer.just_finished(){
          //执行一些操作
        }
    }
}
```

### 4.3.3 时间相关的变量

Bevy的示例中充满了这种用法，我们可以获得`Time`，然后利用`Time`来更新某些变量。例如我们可以通过`delta_secs`方法获得每帧相隔的时间，然后乘以系数并不断累加到某个位置，这可以做到让该变量随着时间不断更新的效果。

```rust
fn animate(mut state: ResMut<AnimationState>, time: Res<Time>) {
    if state.current >= state.max || state.current <= state.min {
        state.speed = -state.speed;
    };
    state.current += state.speed * time.delta_secs();
}
```

