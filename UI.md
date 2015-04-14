UI

GhostCat的UI组件借鉴了FLEX的命名方式，组件部分并没有多少可讨论的，new出来然后设置属性并使用就可以。下面要说的是两个需要注意的地方，一个是布局，一个是皮肤。

布局

我本来是想完全不做布局的。如果需要复杂布局的话，就不应该用我这个东西，在拥有复杂布局的同时体积也不可能小得下来。GhostCat的布局使用的是LayoutUtil这个静态类的方法，以及用layout类布局器，它是一个完全和容器没耦合的东西。
LayoutUtil里有一些布局方法，直接调用。与其他的不同，它并不要求注册点一定是0,0。
Layout类则是对容器的管理类，便可对指定的容器内对象自动调用LayoutUtil布局。

虽然并不想做布局，但最后还是做了一个View类，简单包装了一次layout。主要的方向是布局子对象，但后来也实现了measureChildren。不过，如果并不需要动态布局的话，建议还是直接执行LayoutUtil的方法为妙，我无法保证布局类的效率。
VBox，Canvas，都是实现了的。但在需要的时候应该使用独立的layout对象手动布局，而不是再套一个容器，那没有必要。View只是对layout的简单包装。

如果真的希望复杂布局，应当使用FLEX或者ASWing。这里的布局类效率并不高，功能也弱。是在必要的时候提供功能用的，省的自己写，而不是推荐使用的东西。

皮肤

GhostCat并没有皮肤管理器，也不存在动态换肤功能。如果想定义皮肤，最简单的办法就是直接修改asset下的默认皮肤fla。如果不想修改库的源文件，可以复制一个fla，在自己的工程重新生成swc并导入，这个新的swc就会自动覆盖掉原来的那个。此外，如果使用的不是默认皮肤，嫌默认皮肤浪费体积的话，也可以创建一个全是空skin的fla生成swc并导入，它会将原来有图形的皮肤替换掉，这就达到了消减多余体积的目的。

默认皮肤实际上就是每个组件类的defaultSkin静态属性，直接修改这个静态属性也是可以的。

但这些并不是设计目的。都只是补充。GhostCat其实从一开始采用的就是替换皮肤的方案：它的所有组件类的第一个属性一定是skin，你只要把皮肤放在这里就好了。GhostCat没有Style，所以所有的表现也都只能通过实际的皮肤。
这个skin属性并不仅仅是设置皮肤。它同时也会接受皮肤的坐标属性，而且会把自己插入到皮肤原来的位置上。可以简单的理解成替换。

诸如：
Var a:Sprite = new AAA();
Var v:GButton = new GButton(a);
V.x = 100;
addChild(v);

这是一般的做法。但这种做法和下面的情况是一样的：

Var a:Sprite = new AAA();
a.x = 100;
addChild(a);
Var v:GButton = new GButton(a);

最下面的一条语句就可以理解成替换，和普通的方法区别就在于创建了以后并不addChild，但这样做皮肤必须已经在舞台之上。如果是在时间线上，则可以省略其他代码直接v:GButton = new GButton(实例名)这样替换。

被替换的皮肤并不一定在第一层，也可以在多层嵌套的任何一个地方。可以通过SearchUtil来查找相应的皮肤并像上面那样进行替换并返回引用。这样对美工提供的FLA就不会有过多的限制：只要名字正确就可以了。
GhostCat对用来替换的皮肤内部有一定的层次要求，因为像List,ScorllBar一类的组件需要让电脑能明白哪些图元各代表的意义是什么。组件的说明上都写明了具体的约定，但约定都并不复杂。此外，也可以通过重新设置fields属性来修改这个约定，即使美工也错了名字也可以进行适应。处理这个约定是唯一麻烦的地方。

如果美工能够正确遵守约定，提供正确的FLA文件，代码这边就会非常简单。以至于，只需要执行一次UIBuilder.buildAll()就可以了。UIBuilder是一个自动执行上面方法批量创建组件的类。

当然，你需要首先建立一个自定义组件，GBase就可以，然后在里面添加几个属性，表示组件里的子组件，并写好它们的类型，名字则必须和皮肤里对应每一项皮肤的名称相同，这样设置皮肤，并执行UIBuilder.buildAll()的时候就会自动进行转换。执行完后所有的属性就都有值了，然后就可以添加事件监听完成逻辑部分。这种方法并不是一直都有效，因为子组件的默认属性并不总是合适的，一种方法就是在buildAll的参数里进行设置，另一种则是放弃buildAll，用build方法一个一个创建，或者连UIBuilder类都不用，还是用SearchUtil找皮肤然后new一个组件替换皮肤并设在属性上的老方法。UIBuilder的目的是提供方便。如果不能提供方便的话，当然也就不该用了。
一般情况的面板靠UIBuilder应当都是可以的。即使是List也一样大多可以通过默认参数创建。需要设置的情况大多都是因为ItemRender，这个值也可以通过buildAll的参数定义。

这个方式有一个优点：因为它可以去替换影片剪辑里的skin，所以和动画不冲突。即使在动画里也能正常显示。但要想正常显示对动画还是有要求的，一个就是包含组件的部分一定要在时间线上始终存在（而不是一直能显示就可以，即使永远不会跳过去的帧上出了空位，一样会导致定义失效），另一个则是不要直接用组件做动画，因为组件是替换式的，它无法替换时间线上的所有实例。取而代之，应当用一个单帧容器包装组件，然后再让这个包装代替组件做动画，这样替换的时候就没问题了。

如果担心嵌套消耗性能，可以将repeater属性改成false,这样组件本身就不会替换皮肤，只是和皮肤建立联系。Text类的还好，Button可就无法接受到鼠标事件了，所以这个只是在必要的时候设置。