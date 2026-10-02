## 3.1 回顾Asset

资产是需要加载到游戏中的资源，通常来自于各种硬盘里的文件，例如图像、模型、材质、字体、音频等等等等。由于这些资源的加载往往需要耗费大量时间，因此Bevy里这些资产的加载往往都是以异步的形式以避免阻塞游戏循环。在Bevy中，我们可以使用`AssetServer`从硬盘里加载资产，使用`Assets<T>`来存储已经加载的各类资产。

与资产相关的内容主要在bevy_asset这个crate中，要使用这些内容，**必须在App上调用其中的[AssetPlugin](https://docs.rs/bevy_asset/0.17.3/bevy_asset/struct.AssetPlugin.html)这个插件才能访问`AssetServer`、`Assets`等类型，这个插件还提供了一些额外的设置，例如指定模式和路径等，该插件已经在DefaultPlugin内，不需要我们再手动安装。**

与Asset相关的类型和结构很多，不过大多数时候我们都不需要和他们打交道，如果仅仅是使用，我们只需要和`AssetServer`还有`Assets<T>`接触就足够了。

### 3.1.1 AssetServer

`AssetServer`作为一种全局资源，可以使用之前我们加载资源的方式以`Res`来获取。默认情况下，加载的资产都相对于项目目录下的assets文件夹，要修改这个默认行为，可以修改`BEVY_ASSET_ROOT`环境变量来指定加载资产的目录。

首先，我们使用`AssetServer`加载了一个图像并获得其句柄，句柄类似于一个对资产的引用计数指针，但能被克隆为强句柄和弱句柄，当不再存在资产的强句柄时，Bevy能够自动将其回收并销毁以释放内存。所以，**为了保证资产的持续存在，必须将句柄存储在一个`Resource`或者`Component`中。**

```rust
#[derive(Resource)]
struct ShareImage {
  handle: Option<Handle<Image>>,
}

fn load_image(asset_server: Res<AssetServer>, mut share_image: ResMut<ShareImage>) {
    let image_handle = asset_server.load("test.png");
    share_image.handle = Some(image_handle);
}
```

由于`AssetServer`返回的是一个句柄并采取异步的方式加载资源，如果你的逻辑中需要判断资源是否加载完成，不能依靠句柄本身存在与否来判断，要实现此功能，可以使用其身上的`get_load_state`方法或`is_loaded_with_dependencies`方法，该方法会返回一个`LoadState`类型的枚举，用来标识加载的阶段。他们的区别将在之后介绍，现在重要的是理解资产的加载是异步的，需要判断是否加载完成才能够使用。

```rust
fn on_asset_event(
  mut commands: Commands,
  asset_server: Res<AssetServer>,
  share_image: Res<ShareImage>,
) {
  match asset_server.get_load_state(&share_image.handle) {
    Some(LoadState::NotLoaded) => {}
    Some(LoadState::Loading) => {}
    Some(LoadState::Loaded) => {
      //在这里使用handle，这时已经加载完成
    }
}
```

### 3.1.2  Assets

**`Assets<T>`** 是一个键值对集合，存储了特定类型 `T` 的所有**实际资产数据**。当`AssetServer`**成功加载资源后**，将会将真正的数据保存在对应的**`Assets<T>`** 中，如果需要获得真正的数据，则需要使用相关的句柄和对应类型的`Assets`

```rust
fn read_image_data(images: ResMut<Assets<Image>>, share_image: Res<ShareImage>) {
    let handle = match &share_image.handle {
        None => return,
        Some(handle) => handle,
    };
    if let Some(image) = images.get(handle) {
        // 现在你有了image的真正数据，可以读取或者修改
        println!("Loaded image size: {:?}", image.size());
    }
}
```

### 3.1.3 自定义资产

如果我们的资产是某种Bevy不支持的格式时，必须手动编写代码和Bevy进行交互来定义我们的**资产类型**、资产的**加载方法**、资产的**设置**以及加载时可能的**错误**。

假如，我们想要声明一个能够加载点云las文件的资产，那么首先需要定义我们的资产数据应该长什么样子。本质上那只是一个点的`Vec`而已，其中每个点都有自己的位置、点的尺寸、以及颜色信息，看起来可能是下面这个样子。

注意到我们使用了`#[derive(Asset)]`来告诉Bevy这是我们的资产。

```rust
//点云资产
#[derive(Asset)]
pub struct PointCloud {
    pub points: Vec<PointCloudData>,
}

//实际的点数据
#[repr(C)]
pub struct PointCloudData {
    pub position: Vec3,
    pub point_size: f32,
    pub color: [f32; 4],
}
```

接着，让我们定义加载时可能出现的一些错误，我们可以使用`thiserror`来快速声明这些错误类型。

```rust
use thiserror::Error;
#[derive(Error, Debug)]
pub enum LasLoaderError {
    #[error("failed to load file: {0}")]
    Io(#[from] std::io::Error),
}
```

之后，让我们定义一些资产的加载设置和加载器，并为我们的加载器实现`AssetLoader`特型，在之前我们介绍过，Bevy中的资产加载是异步的，因此需要使用`async`声明`load`方法。这里的代码没什么神奇的，但值得一提的是这里的 `Reader`读取的是二进制数据，需要使用一个`Vec<u8>`来作为缓冲区存储这些字节数据。

```rust
//在加载时我们可以额外传递一个配置以便动态的控制加载过程，但是在这里我们不需要这些
pub struct LasLoaderSettings{}

//我们的加载器
pub struct LasLoader {}

impl AssetLoader for LasLoader {
    type Asset = PointCloud;
    type Settings = LasLoaderSettings;
    type Error = LasLoaderError;

    async fn load(
        &self,
        reader: &mut dyn bevy_asset::io::Reader,
        _settings: &Self::Settings,
        _load_context: &mut LoadContext<'_>,
    ) -> Result<PointCloud, Self::Error> {
      let mut bin_data = Vec::new();
      reader.read_to_end(&mut bin_data).await?;
      //在这里编写真正加载数据的逻辑
      //let points = ..... 
      //然后返回一个资产
      Ok(PointCloud { points })
  }
```

最后，让我们在`App`中注册这些资产和相应的加载器。

```rust
fn main() {
  App::new()
    .add_plugins(DefaultPlugins)
  	//通过这两个方法注册相应的加载器和资产类型
    .init_asset_loader::<LasLoader>()
    .init_asset::<PointCloud>()
    .add_systems(Startup, load_pointcloud)
    .run();
}

//现在，我们应该能够直接使用这些资产类型了
fn load_pointcloud(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
){
  let point_cloud_handler = asset_server.load::<PointCloud>("pointCloud.las");
}
```

## 3.2 资产加载流程

在 Bevy 引擎中，`AssetReader`、`AssetLoader`、`AssetSaver` 和 `AssetWriter` 都是资产系统的重要组成部分，它们各自承担着不同的职责。

### 3.2.1 AssetLoader

`AssetLoader`是我们主要用于加载资产的核心类型，在第一章的自定义资产中，我们做了如下几件事情：

1. 使用`Asset`指令定义一个`PointCloud`资产
2. 定义对应的`LasLoader`、`LasLoaderError`、`LasLoaderSettings`
3. 为`LasLoader`实现异步的`load`方法，并从中返回一个`Asset`

通过这些操作，我们便可以在系统中这样使用我们自定义的加载器以方便的来实现自定义类型的文件加载，并且，我们还可以通过`LasLoaderSettings`来控制加载的行为。

```rust
fn main() {
  App::new()
    .add_plugins(DefaultPlugins)
  	//通过这两个方法注册相应的加载器和资产类型
    .init_asset_loader::<LasLoader>()
    .init_asset::<PointCloud>()
  	.add_systems(Startup, load_pointcloud)
    .run();
}

//现在，我们应该能够直接使用这些资产类型了
fn load_pointcloud(
    mut commands: Commands,
    asset_server: Res<AssetServer>,
){
  //我们不再简单的使用load方法，而是使用load_with_settings方法来指定setting参数
  let point_cloud_handler = asset_server.load_with_settings::<PointCloud>(
    "pointCloud.las"，
    |settings: &mut LasLoaderSettings| {
         //我们可以对这个默认对象做一些更改
    });
}
```

### 3.2.2 AssetReader

在为`LasLoader`实现异步的`load`方法时，你可能已经注意到了有一个`Reader`参数，他是一个类型擦除的`bevy_asset::io::Reader`特型对象，用于异步地将字节数据读取到缓冲区中。而这个特型对象，正是`AssetReader`的`read`方法返回的。

```rust
impl AssetLoader for LasLoader {
    type Asset = PointCloud;
    type Settings = LasLoaderSettings;
    type Error = LasLoaderError;

    async fn load(
        &self,
        reader: &mut dyn bevy_asset::io::Reader,
        _settings: &Self::Settings,
        _load_context: &mut LoadContext<'_>,
    ) -> Result<PointCloud, Self::Error> {
      let mut bin_data = Vec::new();
      reader.read_to_end(&mut bin_data).await?;
      //在这里编写真正加载数据的逻辑
      //let points = ..... 
      //然后返回一个资产
      Ok(PointCloud { points })
  }
```

`AssetReader`和`AssetLoader`都用于资产的加载，但是他们负责的层级是完全不同的。**`AssetReader`为不同的平台提供了一个统一的、异步的加载方式，使得我们的游戏能够在每个平台上以相同的方式读取资产；而`AssetLoader`负责将数据解析成正确的格式，以供程序的正确使用。**

大多数情况下，我们可以直接使用内置的`AssetReader`不需要对其进行自定义，但如果你需要在这个环节进行一些操作，那么Bevy也提供了相应的方法让我们能够重写我们自己的`Reader`。详细的细节可以查看[文档](https://doc.qu1x.dev/bevy_trackball/bevy_asset/io/trait.AssetReader.html#tymethod.read)和[示例](https://github.com/bevyengine/bevy/blob/main/examples/asset/custom_asset_reader.rs)。

### 3.2.3 AssetSaver与AssetWriter

理解了`AssetReader`与`AssetLoader`的关系，我们可以猜到应该还有两个用于保存资产到本地的类型，这些类型就是`AssetWriter`与`AssetSaver`，他们的关系是类似的：`AssetWriter`提供了一个统一的、异步的写入方式，将我们的资产写入到文件系统中，而`AssetSaver`则负责将数据转换成需要保存的格式。二者的具体使用方法可以查看[文档](https://doc.qu1x.dev/bevy_trackball/bevy_asset/saver/trait.AssetSaver.html)，需要为`AssetSaver`实现一个`save`方法，该方法和`load`非常相似，这里不再赘述。

## 3.3 资产加载

### 3.3.1 内嵌资产

有些时候，我们希望将资产打包进入二进制程序中，然后在程序中直接读取这些资产而不是从硬盘里加载。例如，我们可能编写了一些wgsl着色器而又不想将这些代码作为文件存储在磁盘里，这时就需要将其直接内嵌在二进制的程序中，不过我们会需要一些额外的手段来告诉Bevy如何读取这些内嵌的资源。

Bevy中使用宏来嵌入资产到程序中并读取，他们是`embedded_asset!`宏和`load_embedded_asset!`宏。

`embedded_asset!`宏接受两个或三个参数，当接受两个参数的时候，第一个是当前`app`的可变引用，第二个是需要内嵌的资源的相对于当前目录的路径。假设我们的项目目录如下，我们现在想要将rock.wgsl内嵌到程序中。

```tex
bevy_rock
├── src
│   ├── render
│   │   ├── rock.wgsl
│   │   └── mod.rs
│   └── lib.rs
└── Cargo.toml
```

在render目录下的mod.rs中，我们可以编写一个插件，然后这样使用`embedded_asset!`宏和`load_embedded_asset!`宏。当使用宏加载后，文件将会位于一个虚拟的目录下，在这里是`embedded://bevy_rock/render/rock.wgsl`（注意到src路径已经被删除）。其中前面的`embedded`被称为`AssetSourceId`，每一类`AssetSourceId`都映射着对应的`AssetReader`和`AssetWriter`（从程序中之间内嵌的数据读取方式和从磁盘的读取方式不同，所以需要保留这个信息告诉Bevy正确的读取方式）。

我们还可以利用`app`上的`register_asset_source`方法注册自己的`AssetSourceId`，将其绑定到某些文件夹以便在加载资产时通过前戳来快速访问资产，详细的方法可以参考[文档](https://doc.qu1x.dev/bevy_trackball/bevy_asset/trait.AssetApp.html#tymethod.register_asset_source)。

```rust
//使用两个参数的版本时，第一个参数是app的可变引用，第二个是资产的路径（相对当前目录）
embedded_asset!(app, "rock.wgsl")
//使用三个参数的版本时，第三个参数是资产的路径，第二个参数是需要移除的路径部分
//因此embedded_asset!(app, "rock.wgsl")和embedded_asset!(app, "/src/", "rock.wgsl")是等效的
embedded_asset!(app, "/examples/rock_stuff/", "rock.wgsl")
```

然后，我们有两种方式可以加载这个资源。

```rust
//如果在当前的模块中加载，我们可以直接使用load_embedded_asset
let shader = load_embedded_asset!(&asset_server, "rock.wgsl");

//如果在其他模块中，我们可以使用asset_server和路径全称
let shader = asset_server.load::<Shader>("embedded://bevy_rock/render/rock.wgsl");
```

在Bevy 0.12之前，你可能会看到名为`load_internal_asset!`的宏，该宏的作用和上面是一样的，不过目前已经被`embedded_asset!`取代，因此不建议继续使用。

### 3.3.2 web资产

另一类比较特殊的资产就是从web上加载的资产，在网络上加载一些内如，我们需要一点额外的支持：引入`WebAssetPlugin`插件并开启http特征。

```rust
use bevy::{asset::io::web::WebAssetPlugin, prelude::*};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(WebAssetPlugin {
            silence_startup_warning: true,
        }))
        .add_systems(Startup, setup)
        .run();
}
```

然后我们就可以像使用普通的文件一样，从web的url里加载这些资产了。

```rust
fn setup(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(Camera2d);
    let url = "https://raw.githubusercontent.com/bevyengine/bevy/refs/heads/main/assets/branding/bevy_bird_dark.png";
    commands.spawn(Sprite::from_image(asset_server.load(url)));
}

```

## 3.4 资产事件

资产会在加载的过程中发出一系列Message，如果你需要对这些事件做出一些响应，在bevy中这些事件是一个AssetEvent的枚举类型，可以看到当资产被添加、更改、移除、加载完成时都会发出一些事件，并且在其中保存了对应的资产的ID。

```rust
pub enum AssetEvent<A: Asset> {
    Added {
        id: AssetId<A>,
    },
    Modified {
        id: AssetId<A>,
    },
    Removed {
        id: AssetId<A>,
    },
    Unused {
        id: AssetId<A>,
    },
    LoadedWithDependencies {
        id: AssetId<A>,
    },
}
```

要响应这些事件，只需要使用特定的`MessageReader`即可。

```rust
fn read_message(mut messages: MessageReader<AssetEvent>) {
  for message in messages.read() {
    //对消息做一些处理
    //...
  }
}
```
