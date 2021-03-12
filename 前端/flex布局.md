### Flex 基本概念

![img](https://pic4.zhimg.com/80/v2-54a0fc96ef4f455aefb8ee4bc133291b_1440w.jpg)

在容器中的每个单元块被称之为 flex item，每个项目占据的主轴空间为 (main size), 占据的交叉轴的空间为 (cross size)。

### Flex 容器

首先，实现 flex 布局需要先指定一个容器，任何一个容器都可以被指定为 flex 布局，这样容器内部的元素就可以使用 flex 来进行布局

```css
.container {
    display: flex | inline-flex;       //可以有两种取值
}
```

分别生成一个块状或行内的 flex 容器盒子。简单说来，如果你使用块元素如 div，你就可以使用 flex，而如果你使用行内元素，你可以使用 inline-flex。

**需要注意的是：当时设置 flex 布局之后，子元素的 float、clear、vertical-align 的属性将会失效。**

**1. flex-direction: 决定主轴的方向(即项目的排列方向)**

```css
.container {
    flex-direction: row | row-reverse | column | column-reverse;
}
```

默认值：row，主轴为水平方向，起点在左端。

<img src="https://pic2.zhimg.com/80/v2-ae8828b8b022dc6f1b28d5b4f7082e91_1440w.jpg" alt="img" style="zoom:50%;" />

row-reverse：主轴为水平方向，起点在右端

<img src="https://pic3.zhimg.com/80/v2-215c8626ac95e97834eddb552cfa148a_1440w.jpg" alt="img" style="zoom:50%;" />

column：主轴为垂直方向，起点在上沿

<img src="https://pic1.zhimg.com/80/v2-33efe75d166a47588e0174d0830eb020_1440w.jpg" alt="img" style="zoom:50%;" />

column-reverse：主轴为垂直方向，起点在下沿

<img src="https://pic2.zhimg.com/80/v2-344757e0fb7eee11e75b127b8485e679_1440w.jpg" alt="img" style="zoom:50%;" />

**2. flex-wrap: 决定容器内项目是否可换行**

默认情况下，项目都排在主轴线上，使用 flex-wrap 可实现项目的换行。

```css
.container {
    flex-wrap: nowrap | wrap | wrap-reverse;
}
```

默认值：nowrap 不换行，即当主轴尺寸固定时，当空间不足时，项目尺寸会随之调整而并不会挤到下一行。

<img src="https://pic4.zhimg.com/80/v2-a590927ad6d83de8840d52a0cf2f0df3_1440w.jpg" alt="img" style="zoom:50%;" />

wrap：项目主轴总尺寸超出容器时换行，第一行在上方

<img src="https://pic2.zhimg.com/80/v2-426949b061e8179aab00cacda8168651_1440w.jpg" alt="img" style="zoom:50%;" />

wrap-reverse：换行，第一行在下方

<img src="https://pic2.zhimg.com/80/v2-91c53ebf744814e1ab60267643866439_1440w.jpg" alt="img" style="zoom:50%;" />

**3.justify-content：定义了项目在主轴的对齐方式。**

```css
.container {
    justify-content: flex-start | flex-end | center | space-between | space-around;
}
```

**建立在主轴为水平方向时测试，即 flex-direction: row**

默认值: flex-start 左对齐

<img src="https://pic1.zhimg.com/80/v2-1bafab80044a7ab2a6198d5937172eb0_1440w.jpg" alt="img" style="zoom:50%;" />

flex-end：右对齐

<img src="https://pic3.zhimg.com/80/v2-8b163809a4c944486a127a7c22eee7b2_1440w.jpg" alt="img" style="zoom:50%;" />

center：居中

<img src="https://pic4.zhimg.com/80/v2-dea82c75d35f532d35a52d1f9c1c762b_1440w.jpg" alt="img" style="zoom:50%;" />

space-between：两端对齐，项目之间的间隔相等，即剩余空间等分成间隙。

<img src="https://pic1.zhimg.com/80/v2-ea4061e0f64dd8d7a1fcb5b0ad6f96a8_1440w.jpg" alt="img" style="zoom:50%;" />

space-around：每个项目两侧的间隔相等，所以项目之间的间隔比项目与边缘的间隔大一倍。

<img src="https://pic1.zhimg.com/80/v2-42a358111a221ff52768bdd55238eb0c_1440w.jpg" alt="img" style="zoom:50%;" />

**4.align-items: 定义了项目在交叉轴上的对齐方式**

```css
.container {
    align-items: flex-start | flex-end | center | baseline | stretch;
}
```

**建立在主轴为水平方向时测试，即 flex-direction: row**

默认值为 stretch 即如果项目未设置高度或者设为 auto，将占满整个容器的高度。

<img src="https://pic2.zhimg.com/80/v2-0cced8789b0d73edf0844aaa3a08926d_1440w.jpg" alt="img" style="zoom:50%;" />

假设容器高度设置为 100px，而项目都没有设置高度的情况下，则项目的高度也为 100px。

flex-start：交叉轴的起点对齐

<img src="https://pic3.zhimg.com/80/v2-26d9e85039beedd78e412459bd436e8a_1440w.jpg" alt="img" style="zoom:50%;" />

flex-end：交叉轴的终点对齐

<img src="https://pic4.zhimg.com/80/v2-8b65ee47605a48ad2947b9ef4e4b01b3_1440w.jpg" alt="img" style="zoom:50%;" />

center：交叉轴的中点对齐

<img src="https://pic3.zhimg.com/80/v2-7bb9d8385273d8ad469605480f40f8f2_1440w.jpg" alt="img" style="zoom:50%;" />

baseline: 项目的第一行文字的基线对齐

<img src="https://pic3.zhimg.com/80/v2-abf7ac4776302ad078986f7cd0dddaee_1440w.jpg" alt="img" style="zoom:50%;" />

### Flex 项目属性：

有六种属性可运用在 item 项目上：

**1. order: 定义项目在容器中的排列顺序，数值越小，排列越靠前，默认值为 0**

```css
.item {
    order: <integer>;
}
```

<img src="https://pic3.zhimg.com/80/v2-d606874ac9c496b3a0e46573c85e4376_1440w.jpg" alt="img" style="zoom:50%;" />

**2. flex-basis: 定义了在分配多余空间之前，项目占据的主轴空间，浏览器根据这个属性，计算主轴是否有多余空间**

```css
.item {
    flex-basis: <length> | auto;
}
```

默认值：auto，即项目本来的大小, 这时候 item 的宽高取决于 width 或 height 的值。

**当主轴为水平方向的时候，当设置了 flex-basis，项目的宽度设置值会失效，flex-basis 需要跟 flex-grow 和 flex-shrink 配合使用才能发挥效果。**

当 flex-basis 值为 0 % 时，是把该项目视为零尺寸的，故即使声明该尺寸为 140px，也并没有什么用。

当 flex-basis 值为 auto 时，则跟根据尺寸的设定值(假如为 100px)，则这 100px 不会纳入剩余空间

**flex-grow: 定义项目的放大比例**

```css
.item {
    flex-grow: <number>;
}
```

默认值为 0，即如果存在剩余空间，也不放大

<img src="https://pic4.zhimg.com/80/v2-5f7898c1f51fa7274a2c0b4a9dfd88c3_1440w.jpg" alt="img" style="zoom:50%;" />

当所有的项目都以 flex-basis 的值进行排列后，仍有剩余空间，那么这时候 flex-grow 就会发挥作用了。

如果所有项目的 flex-grow 属性都为 1，则它们将等分剩余空间。(如果有的话)

如果一个项目的 flex-grow 属性为 2，其他项目都为 1，则前者占据的剩余空间将比其他项多一倍。

当然如果当所有项目以 flex-basis 的值排列完后发现空间不够了，且 flex-wrap：nowrap 时，此时 flex-grow 则不起作用了，这时候就需要接下来的这个属性

 **flex-shrink: 定义了项目的缩小比例**

```css
.item {
    flex-shrink: <number>;
}
```

默认值: 1，即如果空间不足，该项目将缩小，负值对该属性无效

<img src="https://pic4.zhimg.com/80/v2-383e97971a7fc8c4f84e6a85406dbcaf_1440w.jpg" alt="img" style="zoom:50%;" />

这里可以看出，虽然每个项目都设置了宽度为 50px，但是由于自身容器宽度只有 200px，这时候每个项目会被同比例进行缩小，因为默认值为 1。

同理可得：

如果所有项目的 flex-shrink 属性都为 1，当空间不足时，都将等比例缩小。

如果一个项目的 flex-shrink 属性为 0，其他项目都为 1，则空间不足时，前者不缩小。

**flex: flex-grow, flex-shrink 和 flex-basis的简写**

```css
.item{
    flex: none | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'> ]
} 
```

flex 的默认值是以上三个属性值的组合。

- 当 flex 取值为一个非负数字，则该数字为 flex-grow 值，flex-shrink 取 1，flex-basis 取 0%，如下是等同的：

  ```css
  .item {flex: 1;}
  .item {
      flex-grow: 1;
      flex-shrink: 1;
      flex-basis: 0%;
  }
  ```

  - 当 flex 取值为 0 时，对应的三个值分别为 0 1 0%

```css
.item {flex: 0;}
.item {
    flex-grow: 0;
    flex-shrink: 1;
    flex-basis: 0%;
}
```

当 flex 取值为一个长度或百分比，则视为 flex-basis 值，flex-grow 取 1，flex-shrink 取 1，有如下等同情况（注意 0% 是一个百分比而不是一个非负数字）

```css
.item-1 {flex: 0%;}
.item-1 {
    flex-grow: 1;
    flex-shrink: 1;
    flex-basis: 0%;
}

.item-2 {flex: 24px;}
.item-2 {
    flex-grow: 1;
    flex-shrink: 1;
    flex-basis: 24px;
}
```

当 flex 取值为两个非负数字，则分别视为 flex-grow 和 flex-shrink 的值，flex-basis 取 0%，如下是等同的

```css
.item {flex: 2 3;}
.item {
    flex-grow: 2;
    flex-shrink: 3;
    flex-basis: 0%;
}
```

































