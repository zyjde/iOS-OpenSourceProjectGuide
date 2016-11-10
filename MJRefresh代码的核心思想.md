![image](https://github.com/Hunter-HYB/iOS-OpenSourceProjectGuide/blob/master/Resource/MJRefresh.png)

上图为MJRefresh项目的项目结构

###MJRefresh代码的核心思想

在MJRefresh中，使用了KVO、runtime、继承、GCD等知识

###核心思想
--
MJRefreshComponent是刷新控件的基类，在MJRefreshComponent添加了KVO监听、prepare方法和placeSubviews方法。

- 当MJRefreshComponent中KVO监听到之后，响应会在MJRefreshHeader和MJRefreshFooter中实现，MJRefreshHeader和MJRefreshFooter其实响应KVO方法，主要就是设置state状态，然后在他们的子类中会分别调用setState方法，根据不同的state状态进行不同的变化

- prepare方法和placeSubviews方法。prepare是设置刷新控件，包括文字、gif图片、风格等等；placeSubviews是调整刷新控件的子控件的位置。在MJRefreshComponent的每一个子类中都会先调用父类对应方法，然后根据自身的特点进行不同实现
 
