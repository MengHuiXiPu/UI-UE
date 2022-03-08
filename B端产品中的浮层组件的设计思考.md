# B端产品中的浮层组件的设计思考

浮层是在页面上方展示的浮层容器，可展示文本、按钮、列表、标签、表单项等内容，在B端产品中有着非常广泛的应用。根据内容和作用，可以分为不同的设计组件。例如Notification，Tooltip，Dialog等等。这些组件都可以看作是页面空间的延展，最近在项目工作中有了一些新的思考，今天我们就来讨论下浮层组件的设计，希望大家能够有新的收获。

主要内容包括以下4部分：

1. 浮层组件的类型
2. 浮层组件3个的特点
3. 几个应用案例
4. 浮层组件应用的注意事项

**01/浮层组件的类型**

说起浮层就离不开“模态”和“非模态”。

简单的理解，“模态”就是用户必须进行交互操作的浮层，否则浮层会一直存在，并且无法进行页面功能操作。例如对话框，这种强制性保证了对话框信息的有效传递，但是对户操作流程造成了比较强的阻断。“非模态”则不需要给出反馈，不影响用户的其他操作，没有强制性，可以说是“来去自如”。

微信PC端图片查看功能中信息提示很好的反映了两者的差异。对于无法查看的图片，采用模态弹窗形式，提醒用户无法查看的原因，用户需要点击确定后，才能继续操作。如果查看到最后一张图片，系统采用非模态的Notatifion组件提醒。组件未消失时，用户也可以回看或其他操作，更加轻量化。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongDAsgdzYxhL5RiaUsUSCEiaoez4wnhNlE4O1cMbPaTUETEBMo7iaZRPUs6nOLAGsLf7tEFAO2qjhMI6A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

具体到组件设计层面，浮层的类型就比较多了。例如AntDesign 设计规范共定义了8种浮层组件，Element 设计规范则定义了9种浮层组件，增加了Messagebox组件，其实就是Dialog 对话框的简易版。具体的差异感兴趣的同学可以自己去查阅一下。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongDAsgdzYxhL5RiaUsUSCEiaoeGuckXQduiaRRL6nDibIia03UvpTfeiamzZiakcYxXhEiaN6vZX2BNK5rF8yg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)▲ Ant Design设计规范

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongDAsgdzYxhL5RiaUsUSCEiaoej8jib72RXDIiaKdotkvhPbOkZ3XEA9ZICcgrmHuia0AKiaXQ9sBArnYTCg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

▲ Element 设计规范



**02/浮层组件的特点**

1）构建独立空间，简化页面信息量

浮层在一定的条件下触发展示，作为一种容器可以形成临时的内容空间。不会占据页面空间，并且可以简化页面的信息量，有助于页面的内容布局。例如常见的数据可视化展示时，重点在于图形展示，详细数据放置在浮窗中展示，从而保证了页面的可视化效果。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongAhfvYuOIUlIngagPA62jhIwnzfLb8pSX3ofHkicf1FANnDicrTnNqT4LibEjiafKASFTL8JvFlCbq1Fw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

▲浮窗展示空间



2）交互轻量化

对于强调操作效率的B端产品，如非必要，需要尽量减少页面的跳转次数，实现当前页面内的流程闭环。

在交互流程上，浮层组件停留在当前页面，相比页面跳转的交互方式效率更高。

在触发机制上，非模态浮层可以实现悬停展示，移走消失，操作更加方便。某些信息反馈类的组件还可以自动消失，最大程度上减少了用户操作。并且非模态浮层同样可以承载按钮、选择器，表单等功能性组件，用户可以就近操作，从而提高效率。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongAhfvYuOIUlIngagPA62jhInlBe832nzia1wvVTGIY3P9uicXM81I64piciaWpo3UzzVtbyicOibcRMyvYQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

▲QQ浏览器设置浮窗



在页面布局方面，浮层也更加灵活，可以出现在页面中间、侧边、上方、下方等各个位置，尺寸可大可小，对于不太复杂的信息都有较好的适应性。

3）实现操作的所见即所得

由于非模态窗口不具有强交互性，一般不占据屏幕中间位置，更多的是跟随功能选项或者页面边缘。这就为功能操作的所见所得提供了便利性，方便用户做出操作决策。例如表格列设置功能，操作后即可实时展示操作结果。或者页面皮肤的设置，用户选择后即可预览效果，方便用户对比选择。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/Z1npGiadongCoextM810P4QBibjbATNDkXJKEXoXArztBzRib2pBNV8ZK6bTxWL9z1csyO1zZZljjUlFSQ9ce0hVQ/640?wx_fmt=gif&wxfrom=5&wx_lazy=1)

▲QQ浏览器皮肤切换

**03/浮层组件的应用案例**

在实际使用场景中，浮层组件主要应用在信息反馈、内容展示和功能操作3个方向，给大家介绍几个案例。

1）预览展示，减少用户的操作成本

Windows11 任务栏悬停程序图标时，显示浮窗预览当前运行的任务，通过图形化的方式，让用户更好的识别任务内容，便于用户做出选择。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongAhfvYuOIUlIngagPA62jhIRSclibLpuo5XvzkF1ibonZ3leRCR0n3lh0hCj7aK5obGFZIhaT0LrZ2Q/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

在淘宝Web端订单页面，商品在未收货状态下，物流状态就成为用户更加关心的信息，悬停显示物流最新状态，用户可以不进入详情页快速获取信息，交互上更加便捷，有效的减少了用户操作成本。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongAhfvYuOIUlIngagPA62jhIBeic912wicceXibPqxNtddk4dPXicgkv6lGicep0Gpex1lSAwvKENbrIq9g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



2）就近原则，快捷操作

在B端产品中，表格信息如何快速分拆内容，查看单个数据的详情信息呢？

如果采用跳转页面，或者表格按钮等形式，都无法带给用户更加流畅的操作体验。通过浮层展示功能选项，就可以实现点对点的查看信息详情。类似于操作系统的右键功能，方便快捷。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongAhfvYuOIUlIngagPA62jhIhTicp1oyxmtwibzf9zXpuKFVDCicnhz67MMYNGaWJ4OzVIs8M7icib4j19A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

同样在微信公众号编辑器中，悬停和选中图片都可以调出图片编辑功能，就近的设计形式，保证了用户的操作效率。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongAhfvYuOIUlIngagPA62jhINF1hdkkGISrjLCRic3FtC1s99kwjPgoccX1SDl6q14dzBia5DmW5waUg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

3）引导作用，辅助功能指向

我们使用Chrome浏览器登录网站时，经常提醒我们保存或者更新密码。这是脱离了网页之外的浏览器自带功能。如果采用模态对话框方式，在屏幕中间弹窗展示确认对话框，会阻断用户的正常操作流程。使用了非模态窗口，并与密码管理功能入口强关联，可以引导用户认知功能入口。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Z1npGiadongAhfvYuOIUlIngagPA62jhIjDQcDDvdh2mtDQPIJzANaNMjNgD1ibRGItBPFGbSiaZRTkcwPRanFGpA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

**04/组件使用注意事项**

对于组件如何使用，各大厂商都给出了较为明确的场景说明。但是规范是死的，如何灵活运用就需要“仁者见仁、智者见智”了。

1）基于用户场景的思考

当我们面对删除功能的时候，基于防错原则，首先想到是增加确认弹窗，这看起来是没有问题的。但是不是最优的解决方案呢？

例如同样是删除网址收藏功能，QQ浏览器和Chrome浏览器给出了两种解决方案。QQ浏览器删除时，增加了确认弹窗，用户确认后可删除。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

▲QQ浏览器确认弹窗

Chrome 浏览器的方案时，顺应用户操作，直接删除内容，显示成功提示的同时，增加了容错的“撤回”按钮。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

▲Chrome浏览器浮窗提醒

我们可以先思考下用户场景，删除是个比较高风险的操作，用户一般只有产生了强烈又明确的意图时，才会主动删除内容。无论是确认弹窗还是撤销功能，都是为了避免用户误操作。所以就要评估用户误操作的几率和误操作后带来的用户损失。就书签而言，用户损失并不大，误删除后再加入收藏就可以了。

**◢** **关于误操作的几率**

QQ浏览器只有选中内容后，才能激活删除按钮，另外还可以通过右键菜单、更多按钮、选择”删除“选项后才能完成操作，其实防错机制做的比较好的，因此误操作的几率比较低。

**◢** **关于操作成本**

误操作几率比较低的情况下，我个人认为Chrome 容错的设计方案更优一些，删除的流程更顺畅，只需要在误操作时撤销就可以了。QQ浏览器确认弹窗的方式操作成本更高，不过好在能够批量删除。在一定程度上解决了频繁弹窗的问题。

这种容错思维在Chrome中其他场景中也有应用，我觉得还是值得借鉴的。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

2）避免滥用

现在各种反馈有点泛滥的趋势，所见即所得的场景下，我觉得并不需要增加反馈信息。例如登录页面成功后会直接跳转至系统界面，登录成功的提示就有点画蛇添足了。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

3）毫无来源的反馈信息

当用户打开页面，没有任何操作就弹出一个提示信息，而且是一闪而过，只会让用户用户一脸疑惑。所以需要注意提示信息的形式和节奏，避免让提示信息成为用户的负担。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

