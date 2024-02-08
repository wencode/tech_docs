# 使用Rust语言和GPU技术实现用户界面的120帧每秒渲染

* 原文：[Leveraging Rust and the GPU to render user interfaces at 120 FPS](https://zed.dev/blog/videogame)

现代显示屏的刷新率在每秒60到120帧之间变化，这意味着应用程序在每帧只有8.33毫秒的时间来把像素推送到屏幕上。这个过程包括更新应用状态、布局UI元素，最后把数据写进帧缓冲区。

这是一个严格的时间限制。如果你有使用Electron构建应用的经验，你可能会发现始终达到这个标准几乎是不可能的。在开发Atom时，我们就有这种感觉：无论我们如何努力，总有阻碍我们准时完成帧的因素。比如，因垃圾回收导致的随机暂停就会让我们错过一个帧。又如，一次成本高昂的DOM重新布局，我们又错过了一个帧。帧率从未稳定，许多原因都超出了我们的控制。

然而，当我们在微优化Atom这个由简单盒子和字形构成的渲染管道时，我们却惊叹于计算机游戏如何能以每秒120帧的恒定速度渲染出美丽而复杂的几何图形。为什么渲染几个<div>标签会比绘制一个三维、逼真的角色要慢得多？

在我们开始构建Zed这个项目时，我们决心打造一个反应速度极快、几乎让人感觉不到的代码编辑器。受到游戏世界的启发，我们意识到实现我们所需性能的唯一方法是自己开发一个UI框架，即GPUI。

![Watch the video](https://zed.dev/video/built-like-a-videogame.webm)
* Zed的渲染方式类似于视频游戏，使我们能够将用户界面中的各个层次分开，并模拟一个3D摄像机在它们周围旋转。
 *

## GPUI：渲染

在我们开始构建Zed时，利用GPU进行任意2D图形渲染仍然是一个研究课题。我们尝试过Patrick Walton的[Pathfinder](https://github.com/servo/pathfinder) crate，但它的速度不足以满足我们的性能目标。

因此，我们重新思考了我们要解决的问题。虽然能够渲染任意图形的库听起来很不错，但实际上我们并不真正需要这样的库来为Zed服务。事实上，大多数2D图形界面主要由几种基本元素组成：矩形、阴影、文本、图标和图像。

我们决定不再专注于通用图形库，而是专注于为我们知道Zed的UI需要渲染的每个特定图形元素编写定制的着色器。通过在CPU上以数据驱动的方式描述每个元素的属性，我们可以把所有繁重的工作交给GPU，让UI元素能够并行绘制。

在接下来的部分中，我将介绍GPUI用于绘制每种元素的技术。

### 绘制矩形
矩形作为图形用户界面的基础构件非常重要。

要理解GPUI中绘制矩形的方式，我们首先需要深入了解一下“符号距离函数”（简称SDF）。顾名思义，SDF是一种根据给定位置返回到某个数学定义对象边缘距离的函数。当位置接近对象时，距离趋近于零，并在穿过其边界时变成负数。

SDF的已知列表非常丰富，这主要得益于Inigo Quilez在这一领域的[开创性工作](https://iquilezles.org/articles/distfunctions)。在他的网站上，你还可以找到一系列技术，这些技术允许对SDF进行变形、组合和重复，以创建最复杂和逼真的3D场景。真的，[去看看吧](https://iquilezles.org/)，它非常惊人。

回到矩形的话题：我们来为矩形推导一个SDF。我们可以通过将想要绘制的矩形置于原点来简化问题。从这里开始，我们可以相对容易地看出这个问题是对称的。换句话说，计算位于四个象限中的一个点的距离等同于计算该点在其他三个象

这意味着我们只需关注矩形的右上部分。以角落为参考点，我们可以区分以下三种情况：

* 情况1)，点位于角落的上方和左方。在这种情况下，点与矩形之间的最短距离是由点到矩形顶边的垂直距离决定的。
* 情况2)，点位于角落的下方和右方。在这种情况下，点与矩形之间的最短距离是由点到矩形右边的水平距离决定的。
* 情况3)，点位于角落的上方和右方。在这种情况下，我们可以使用勾股定理来确定角落和点之间的距离。
* 情况3可以泛化处理，以涵盖前两种情况，只要我们不允许距离向量出现负分量。

我们刚刚概述的规则足以绘制一个简单的矩形，并且稍后在这篇文章中，我们将阐述这些规则如何转化为GPU代码。但在我们进行下一步之前，我们可以进行一个简单的观察，这使我们能够将这些规则扩展到计算带有圆角的矩形的SDF上！

注意在上述情况3)中，有无限多个点与角落保持相同的距离。事实上，这些点并非随机点，它们构成了一个圆，其原点位于角落，半径等于这个距离。

当我们逐渐远离直角矩形时，边界开始变得更加光滑。这就是绘制圆角的关键洞察：给定一个期望的圆角半径，我们可以缩小原始矩形的尺寸，计算出到该点的距离，然后从计算出的距离中减去圆角半径。

将矩形SDF迁移到GPU是一个非常直观的过程。简要回顾一下，经典的GPU管线包括顶点着色器和片段着色器。

顶点着色器的作用是将任意输入数据映射到三维空间中的点上，每三个点定义了我们想在屏幕上绘制的一个三角形。接着，对于由顶点着色器生成的三角形内的每个像素，GPU会调用片段着色器，该着色器负责给定像素的颜色赋值。

在我们的情况下，我们用顶点着色器来定义我们想在屏幕上绘制的形状的边界框，使用两个三角形。我们不一定会填充这个框内的每个像素，这是由片段着色器来决定的，我们接下来将讨论这一点。

以下代码使用Metal着色器语言编写，旨在用于[实例化渲染](https://metalbyexample.com/instanced-rendering/)，可以通过单一绘制调用将多个矩形绘制到屏幕上：

```metal
struct RectangleFragmentInput {
    float4 position [[position]];
    float2 origin [[flat]];
    float2 size [[flat]];
    float4 background_color [[flat]];
    float corner_radius [[flat]];
};
 
vertex RectangleFragmentInput rect_vertex(
    uint unit_vertex_id [[vertex_id]],
    uint rect_id [[instance_id]],
    constant float2 *unit_vertices [[buffer(GPUIRectInputIndexVertices)]],
    constant GPUIRect *rects [[buffer(GPUIRectInputIndexRects)]],
    constant GPUIUniforms *uniforms [[buffer(GPUIRectInputIndexUniforms)]]
) {
    float2 position = unit_vertex * rect.size + rect.origin;
    // `to_device_position` translates the 2D coordinates into clip space.
    float4 device_position = to_device_position(position, viewport_size);
    return RectangleFragmentInput {
      device_position,
      rect.origin,
      rect.size,
      rect.background_color,
      rect.corner_radius
    };
}
```
为了决定给边界框内的每个像素分配什么颜色，片段着色器会计算每个像素到矩形的距离，并只有当像素位于矩形的边界内（也就是说，当距离为零时）才对像素进行填充：

```metal
float rect_sdf(
    float2 absolute_pixel_position,
    float2 origin,
    float2 size,
    float corner_radius
) {
    float2 half_size = size / 2.;
    float2 rect_center = origin + half_size;
 
    // Change coordinate space so that the rectangle's center is at the origin,
    // taking advantage of the problem's symmetry.
    float2 pixel_position = abs(absolute_pixel_position - rect_center);
 
    // Shrink rectangle by the corner radius.
    float2 shrunk_corner_position = half_size - corner_radius;
 
    // Determine the distance vector from the pixel to the rectangle corner,
    // disallowing negative components to simplify the three cases.
    float2 pixel_to_shrunk_corner = max(float2(0., 0.), pixel_position - shrunk_corner_position);
 
    float distance_to_shrunk_corner = length(pixel_to_shrunk_corner);
 
    // Subtract the corner radius from the calculated distance to produce a
    // rectangle having the desired size.
    float distance = distance_to_shrunk_corner - corner_radius;
 
    return distance;
}
 
fragment float4 rect_fragment(RectangleFragmentInput input [[stage_in]]) {
    float distance = rect_sdf(
        input.position.xy,
        input.origin,
        input.size,
        input.corner_radius,
    );
    if (distance > 0.0) {
        return float4(0., 0., 0., 0.);
    } else {
        return input.background_color;
    }
}
```

### 投影
为了在GPUI中实现投影效果，我们采用了Figma联合创始人Evan Wallace开发的一种[技术](https://madebyevan.com/shaders/fast-rounded-rectangle-shadows/)。为了完整性，我会在这里概述博客文章的内容，但是阅读原文绝对是值得的。

通常在应用程序中，投影是通过高斯模糊来实现的。对于每一个输出像素，高斯模糊是所有周围输入像素的加权平均结果，每个像素的权重根据像素距离的远近以高斯曲线的方式递减。

[视频:对Zed标志应用高斯模糊。](https://zed.dev/img/post/videogame/gaussian.webm)

如果我们进入连续领域，我们可以将上述过程视为输入信号（在离散情况下，即图像的像素）与高斯函数（在离散情况下，即表示高斯概率分布值的矩阵）的卷积。卷积是一种特殊的数学运算符，通过对两个函数的乘积进行积分来产生一个新的函数，其中一个函数（不论哪一个）在y轴上被反转。直观上，这就像我们在图像上滑动高斯曲线，为每个像素计算一个移动的、加权的平均值，根据高斯曲线对周围像素的权重进行取样。

高斯模糊的一个有趣特点是它们是可分的。也就是说，模糊可以分别沿x轴和y轴应用，而得到的输出像素与在两个维度上应用单一模糊的结果是一样的。

在矩形的情况下，存在一个不需要采样相邻像素就可以绘制其模糊版本的封闭形式解决方案。这是因为矩形也是可分的，可以表示为两个Boxcar函数的交集，每个维度各一个.

高斯与阶跃函数的卷积等价于高斯的积分，这得到了误差函数（也称为erf）。因此，生成一个模糊的直角矩形实际上是对每个维度分别进行模糊处理，然后将这两个结果相交：

```metal
float rect_shadow(float2 pixel_position, float2 origin, float2 size, float sigma) {
    float2 bottom_right = origin + size;
    float2 x_distance = float2(pixel_position.x - origin.x, pixel_position.x - bottom_right.x);
    float2 y_distance = float2(pixel_position.y - origin.y, pixel_position.y - bottom_right.y);
    float2 integral_x = 0.5 + 0.5 * erf(x_distance * (sqrt(0.5) / sigma));
    float2 integral_y = 0.5 + 0.5 * erf(y_distance * (sqrt(0.5) / sigma));
    return (integral_x.x - integral_x.y) * (integral_y.x - integral_y.y);
}
```

然而，像上面那样的封闭形式解决方案并不适用于圆角矩形与高斯进行二维卷积的情况，因为圆角矩形的公式不是可分离的。Evan Wallace的近似方法的聪明之处在于，他沿着一个轴执行了封闭形式的精确卷积，然后沿着另一个轴手动多次滑动高斯函数：

```metal
float blur_along_x(float x, float y, float sigma, float corner, float2 half_size) {
    float delta = min(half_size.y - corner - abs(y), 0.);
    float curved = half_size.x - corner + sqrt(max(0., corner * corner - delta * delta));
    float2 integral = 0.5 + 0.5 * erf((x + float2(-curved, curved)) * (sqrt(0.5) / sigma));
    return integral.y - integral.x;
}
 
float shadow_rounded(float2 pixel_position, float2 origin, float2 size, float corner_radius, float sigma) {
    float2 half_size = size / 2.;
    float2 center = origin + half_size;
    float2 point = pixel_position - center;
 
    float low = point.y - half_size.y;
    float high = point.y + half_size.y;
    float start = clamp(-3. * sigma, low, high);
    float end = clamp(3. * sigma, low, high);
 
    float step = (end - start) / 4.;
    float y = start + step * 0.5;
    float alpha = 0.;
    for (int i = 0; i < 4; i++) {
        alpha += blur_along_x(point.x, point.y - y, sigma, corner_radius, half_size) * gaussian(y, sigma) * step;
        y += step;
    }
 
    return alpha;
}
```

### 文本渲染
对于像Zed这样文本密集型的应用程序来说，高效渲染字形至关重要。同时，生成与目标操作系统外观和感觉相匹配的文本也同样重要。要理解我们如何在GPUI中解决这两个问题，我们需要了解文本整形和字体光栅化的工作原理。

文本整形是指在给定字符序列和字体的情况下确定应该渲染哪些字形以及它们应该放在哪里的过程。有几个开源整形引擎，操作系统通常也提供类似的API（例如，在macOS上是CoreText）。通常认为整形是相当昂贵的过程，尤其是处理像阿拉伯语或梵文这样本质上排版更为困难的语言时。

关于这个问题的一个关键观察是，文本在帧之间通常不会发生太大的变化。例如，编辑一行代码不会影响周围的行，因此再次整形这些行将是不必要的成本。

因此，GPUI使用操作系统的API来执行整形（这保证了文本与其他本地应用程序的外观保持一致），并维护一个文本-字体对到整形字形的缓存。当某段文本第一次整形时，它会被插入到缓存中。如果后续的帧包含相同的文本-字体对，那么这些整形的字形将被重用。相反，如果文本-字体对在后续的帧中消失，它会从缓存中被删除。这样就分摊了整形的成本，并且只针对从一帧到另一帧发生变化的文本。

另一方面，字体光栅化指的是将字形的向量表示转换成像素的过程。实现光栅化的方法有几种，包括传统的CPU光栅化器，如操作系统所提供的（例如，在macOS上是CoreText）或FreeType，以及一些较新的研究项目，这些项目主要在GPU上利用计算着色器进行（例如，Pathfinder、Forma或Vello）。

然而，正如前面提到的，我们对GPUI的设想是，通过针对特定原语编写着色器，而不是依赖一个能够渲染任意向量图形的通用引擎，我们能够实现最大化的性能。特别是对于文本，我们的目标是渲染出大量的静态内容，这些内容无需交互式变换，且与平台的原生视觉风格匹配。此外，需要渲染的字形集是有限的，并且可以非常有效地进行缓存，因此在CPU上进行渲染实际上并不会成为性能瓶颈。

[](https://zed.dev/img/post/videogame/texture-atlas.png)
* Zed产生的字形截图 *

就像处理文本整形一样，我们让操作系统来处理字形的光栅化，确保文本能够与其他原生应用完美匹配。特别地，我们仅对字形的alpha组件（即不透明度）进行光栅化：关于这一点我们稍后将详细说明。实际上，我们为每个字形渲染了多达16种不同的变体，以应对子像素定位的需求，因为CoreText会微调字形的抗锯齿效果，让它们在X和Y方向上看起来有细微的位移。

随后，这些像素被缓存到一个图集中，这是一个长期存储在GPU上的纹理。每个字形在图集中的位置存储在CPU上，并且利用etagere提供的二维装箱算法，以最小化空间的方式准确地定位字形。

最终，利用之前计算的文本整形信息，这些字形被组合起来，构成应用程序想要渲染的原始文本。

上述过程通过一个单独的实例化绘制调用完成，该调用指定了字形的目标位置及其在图集中的位置：

```metal
typedef struct {
  float2 target_origin;
  float2 atlas_origin;
  float2 size;
  float4 color;
} GPUIGlyph;
```
注意到GPUIGlyph如何允许指定字形的颜色。这就是我们之前仅利用其alpha通道来光栅化字形的原因。通过只存储字形的不透明度，我们可以用一个简单的乘法操作填充任何我们想要的颜色，这样就避免了在图集中为每种使用的颜色存储一个字形的副本。

```metal
struct GlyphFragmentInput {
    float4 position [[position]];
    float2 atlas_position;
    float4 color [[flat]];
};
 
vertex GlyphFragmentInput glyph_vertex(
    uint unit_vertex_id [[vertex_id]],
    uint glyph_id [[instance_id]],
    constant float2 *unit_vertices [[buffer(GPUIGlyphVertexInputIndexVertices)]],
    constant GPUIGlyph *glyphs [[buffer(GPUIGlyphVertexInputIndexGlyphs)]],
    constant GPUIUniforms *uniforms [[buffer(GPUIGlyphInputIndexUniforms)]]
) {
    float2 unit_vertex = unit_vertices[unit_vertex_id];
    GPUIGlyph glyph = glyphs[glyph_id];
    float2 position = unit_vertex * glyph.size + glyph.origin;
    float4 device_position = to_device_position(position, uniforms->viewport_size);
    float2 atlas_position = (unit_vertex * glyph.size + glyph.atlas_origin) / uniforms->atlas_size;
 
    return GlyphFragmentInput {
        device_position,
        atlas_position,
        glyph.color,
    };
}
 
fragment float4 glyph_fragment(
    GlyphFragmentInput input [[stage_in]],
    texture2d<float> atlas [[ texture(GPUIGlyphFragmentInputIndexAtlas) ]]
) {
    constexpr sampler atlas_sampler(mag_filter::linear, min_filter::linear);
    float4 color = input.color;
    float4 sample = atlas.sample(atlas_sampler, input.atlas_position);
    color.a *= sample.a;
    return color;
}
```

值得注意的是，利用字形图集来组合文本的性能几乎达到了GPU的带宽极限，因为我们实际上是在将字节从一个纹理复制到另一个纹理，并在过程中执行乘法操作。速度再快也不过如此了。

## GPUI中的Element trait

至此，我们讨论了渲染实现的底层细节。然而，在使用GPUI创建应用程序时，这些细节问题都被完全抽象化了。相反，当框架的用户需要创建无法通过现有元素的组合来表达的新的图形界面元素时，他们将会使用Element特征进行交互：

```rust
pub trait Element {
    fn layout(&mut self, constraint: SizeConstraint) -> Size;
    fn paint(&mut self, origin: (f32, f32), size: Size, scene: &mut Scene);
}
```

GPUI中的布局设计深受[Flutter](https://flutter.dev/)启发。具体而言，元素以树状结构嵌套，约束条件从上而下传递，而尺寸信息则从下而上流动。约束定义了给定元素可以拥有的最小和最大尺寸：

```rust
pub struct SizeConstraint {
    pub min: Size,
    pub max: Size,
}
 
pub struct Size {
    pub width: f32,
    pub height: f32,
}
```

根据元素的特征，布局方法可以决定为其子元素生成新的一组约束，以适应元素增加的任何额外视觉细节。例如，如果某个元素想在其子元素周围画一个1像素的边框，它就应该把由父元素提供的最大宽度和高度各减少1像素，并把这个减少后的约束传递给子元素：

```rust
pub struct Border {
    child: Box<dyn Element>,
    thickness: f32,
    color: Color,
}
 
impl Element for Border {
    fn layout(&mut self, mut constraint: SizeConstraint) -> Size {
        constraint.max.x -= self.thickness;
        constraint.max.y -= self.thickness;
        let (width, height) = self.child.layout(constraint);
        Size {
            width: width + self.thickness,
            height: height + self.thickness,
        }
    }
 
    fn paint(&mut self, origin: (f32, f32), size: Size, scene: &mut Scene) {
        // ...
    }
}
```

一旦确定了元素的大小，就可以进行元素树的绘制了。绘制过程包括按照布局安排元素的子项位置以及绘制元素本身的视觉元素。这个过程结束时，所有元素都会将它们的图形组件推送到一个平台无关的Scene结构体中，这个结构体收集了上文渲染部分描述的原语：

```rust
pub struct Scene {
    layers: Vec<Layer>
}
 
struct Layer {
    shadows: Vec<Shadow>,
    rectangles: Vec<Rectangle>,
    glyphs: Vec<Glyph>,
    icons: Vec<Icon>,
    image: Vec<Image>
}
```

渲染器在绘制原语时按照特定的顺序进行。它开始于绘制所有的阴影，然后是所有的矩形，接下来是所有的字形等等。这样做可以防止某些原语被绘制在其他原语之前：比如，矩形就永远不会被渲染在字形之上。

但是，在某些情况下，这种行为并不是我们所期望的。例如，应用可能希望在按钮前绘制一个工具提示元素，因此需要将工具提示的背景渲染在按钮文本之上。为此，元素可以向场景中推送一个Layer，确保它们的图形元素被渲染在其父元素之上。

GPUI也支持创建新的堆叠上下文，这允许以一种类似于[画家算法](https://en.wikipedia.org/wiki/Painter's_algorithm)的方式进行任意的z-index定位。

继续上述边框的例子，paint方法应先推送一个Rectangle，绘制它想要的边框，然后将子元素定位，以避免与新绘制的边框重叠：

```rust
impl Element for Border {
    fn layout(&mut self, mut constraint: SizeConstraint) -> Size {
        // ...
    }
 
    fn paint(&mut self, origin: (f32, f32), mut size: Size, scene: &mut Scene) {
        scene.push_rectangle(Rectangle {
            origin,
            size,
            border_color: self.color,
            border_thickness: self.thickness,
        });
 
        let (mut child_x, mut child_y) = origin;
        child_x += self.thickness;
        child_y += self.thickness;
 
        let mut child_size = size;
        child_size.width -= self.thickness;
        child_size.height -= self.thickness;
 
        self.child.paint((child_x, child_y), child_size, scene);
    }
}
```

GPUI提供了众多开箱即用的元素，以提供丰富的视觉体验。其中一些元素仅改变其子元素的位置和大小（例如，实现弹性盒模型的Flex），而另一些元素则添加了新的图形功能（例如，Label元素可以用指定的样式渲染文本）。

## 结论
这篇文章快速介绍了GPUI的渲染引擎及其如何被封装成一个API，该API允许对布局和绘制进行封装。GPUI的另一个重要功能是响应用户事件，维护应用状态，并将该状态转换为元素。

我们期待在未来的文章中继续探讨这些话题，敬请关注，以获取更多即将发布的信息！
