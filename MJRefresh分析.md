##MJRefresh分析
![image](https://github.com/Hunter-HYB/iOS-OpenSourceProjectGuide/blob/master/Resource/MJRefresh.png)

我们开始使用MJRefresh的时候，往往都是几行代码就调用了，例如：

    MJRefreshNormalHeader *header = [MJRefreshNormalHeader headerWithRefreshingBlock:^{
        [self reloadData];
        [self.tableView.mj_header endRefreshing];
    }];
    self.tableView.mj_header = header;

###现在来看看，上面的几行代码到底经历着怎样的实现？              
上面的代码，是创建了一个`MJRefreshNormalHeader `的对象，然后将它赋给了`self.tableView.mj_header`，`mj_header`是什么呢？我们找到`UIScrollView+MJRefresh.h`文件，可以看到这是一个分类
	
	#import <UIKit/UIKit.h>
	#import "MJRefreshConst.h"

	@class MJRefreshHeader, MJRefreshFooter;

	@interface UIScrollView (MJRefresh)
	/** 下拉刷新控件 */
	@property (strong, nonatomic) MJRefreshHeader *mj_header;
	@property (strong, nonatomic) MJRefreshHeader *header MJRefreshDeprecated("使用mj_header");
	/** 上拉刷新控件 */
	@property (strong, nonatomic) MJRefreshFooter *mj_footer;
	@property (strong, nonatomic) MJRefreshFooter *footer MJRefreshDeprecated("使用mj_footer");

	#pragma mark - other
	- (NSInteger)mj_totalDataCount;
	@property (copy, nonatomic) void (^mj_reloadDataBlock)(NSInteger totalDataCount);
	@end
	
作者利用runtime的技巧给这个分类添加了五个属性和一个方法，然后将封装好的刷新控件添加给UIScrollview   
如果对runtime不熟悉的，可以查看我之前写的一篇文章 [runtime简要分析](https://github.com/Hunter-HYB/runtime)

	- (void)setMj_header:(MJRefreshHeader *)mj_header
	{
    	if (mj_header != self.mj_header) {
        // 删除旧的，添加新的
        [self.mj_header removeFromSuperview];
        [self insertSubview:mj_header atIndex:0];
        
        // 存储新的
        [self willChangeValueForKey:@"mj_header"]; // KVO
        objc_setAssociatedObject(self, &MJRefreshHeaderKey,
                                 mj_header, OBJC_ASSOCIATION_ASSIGN);
        [self didChangeValueForKey:@"mj_header"]; // KVO
    	}
	}

	- (void)setMj_footer:(MJRefreshFooter *)mj_footer
	{
    	if (mj_footer != self.mj_footer) {
        // 删除旧的，添加新的
        [self.mj_footer removeFromSuperview];
        [self insertSubview:mj_footer atIndex:0];
        
        // 存储新的
        [self willChangeValueForKey:@"mj_footer"]; // KVO
        objc_setAssociatedObject(self, &MJRefreshFooterKey,
                                 mj_footer, OBJC_ASSOCIATION_ASSIGN);
        [self didChangeValueForKey:@"mj_footer"]; // KVO
    	}
	}
所以其实现在可以理解，`self.tableView.mj_header = header;`其实就是给tableview添加一个头部的刷新控件.而增加的属性`MJRefreshHeader `就是刚才创建的`MJRefreshNormalHeader `的基类。`MJRefreshHeader `继承于`MJRefreshComponent`, `MJRefreshComponent`是整个刷新控件的基类。

我们创建了`MJRefreshNormalHeader `的对象，直接调用了一个类方法`headerWithRefreshingBlock`,这个方法是它父类`MJRefreshHeader `的一个方法

	“MJRefreshHeader.m”文件
	+ (instancetype)headerWithRefreshingBlock:(MJRefreshComponentRefreshingBlock)refreshingBlock
	{
    	MJRefreshHeader *cmp = [[self alloc] init];
    	cmp.refreshingBlock = refreshingBlock;
    	return cmp;
	}
	
此方法是为了创建一个`MJRefreshHeader `对象，在创建对象init的时候，又会调用到`MJRefreshHeader`的父类`MJRefreshComponent `的方法

	@implementation MJRefreshComponent
	#pragma mark - 初始化
	- (instancetype)initWithFrame:(CGRect)frame
	{
    	if (self = [super initWithFrame:frame]) {
        	// 准备工作
        	[self prepare];
        
        	// 默认是普通状态
        	self.state = MJRefreshStateIdle;
    	}
    	return self;
	}
	
注意：MJRefreshComponent 类中的prepare方法，会被它的子类都进行调用，每个字类的prepare方法，都会调用父类中的prepare方法，然后增加自己特有的执行操作。
执行完init方法，最后会返回一个MJRefreshNormalHeader对象，然后添加给self.scrollview，添加上去后，便会开始执行`MJRefreshComponent `中的`- (void)willMoveToSuperview:(UIView *)newSuperview`方法

	- (void)willMoveToSuperview:(UIView *)newSuperview
	{
    	[super willMoveToSuperview:newSuperview];
    
    	// 如果不是UIScrollView，不做任何事情
    	if (newSuperview && ![newSuperview isKindOfClass:[UIScrollView class]]) return;
    
    	// 旧的父控件移除监听
    	[self removeObservers];
    
    	if (newSuperview) { // 新的父控件
        	// 设置宽度
        	self.mj_w = newSuperview.mj_w;
        	// 设置位置
        	self.mj_x = 0;
        
        	// 记录UIScrollView
        	_scrollView = (UIScrollView *)newSuperview;
        	// 设置永远支持垂直弹簧效果
        	_scrollView.alwaysBounceVertical = YES;
        	// 记录UIScrollView最开始的contentInset
        	_scrollViewOriginalInset = _scrollView.contentInset;
        
        	// 添加监听
        	[self addObservers];
    	}
	}

监听了三个值，分别是UIScrollView的ContentOffSet、ContentSize、滑动手势的状态

	#pragma mark - KVO监听
	- (void)addObservers
	{
    	NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    	[self.scrollView addObserver:self forKeyPath:MJRefreshKeyPathContentOffset options:options context:nil];
    	[self.scrollView addObserver:self forKeyPath:MJRefreshKeyPathContentSize options:options context:nil];
    	self.pan = self.scrollView.panGestureRecognizer;
    	[self.pan addObserver:self forKeyPath:MJRefreshKeyPathPanState options:options context:nil];
	}
	
利用KVO监听到之后，都会响应相应的didChange方法，比如我们下拉刷新，下拉必然会让contentOffSet发生变化，必然会响应对应的方法：

	MJRefreshHeader文件
	- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change
	{
    	[super scrollViewContentOffsetDidChange:change];
    
    	// 在刷新的refreshing状态
    	if (self.state == MJRefreshStateRefreshing) {
        	if (self.window == nil) return;
        
        	// sectionheader停留解决
        	CGFloat insetT = - self.scrollView.mj_offsetY > _scrollViewOriginalInset.top ? - self.scrollView.mj_offsetY : _scrollViewOriginalInset.top;
        	insetT = insetT > self.mj_h + _scrollViewOriginalInset.top ? self.mj_h + _scrollViewOriginalInset.top : insetT;
        	self.scrollView.mj_insetT = insetT;
        
        	self.insetTDelta = _scrollViewOriginalInset.top - insetT;
        	return;
    	}
    
    	// 跳转到下一个控制器时，contentInset可能会变
     	_scrollViewOriginalInset = self.scrollView.contentInset;
    
    	// 当前的contentOffset
    	CGFloat offsetY = self.scrollView.mj_offsetY;
    	// 头部控件刚好出现的offsetY
    	CGFloat happenOffsetY = - self.scrollViewOriginalInset.top;
    
    	// 如果是向上滚动到看不见头部控件，直接返回
    	// >= -> >
    	if (offsetY > happenOffsetY) return;
    
    	// 普通 和 即将刷新 的临界点
    	CGFloat normal2pullingOffsetY = happenOffsetY - self.mj_h;
    	CGFloat pullingPercent = (happenOffsetY - offsetY) / self.mj_h;
    
    	if (self.scrollView.isDragging) { // 如果正在拖拽
        	self.pullingPercent = pullingPercent;
        	if (self.state == MJRefreshStateIdle && offsetY < normal2pullingOffsetY) {
            	// 转为即将刷新状态
            	self.state = MJRefreshStatePulling;
        	} else if (self.state == MJRefreshStatePulling && offsetY >= normal2pullingOffsetY) {
            	// 转为普通状态
            	self.state = MJRefreshStateIdle;
        	}
    	} else if (self.state == MJRefreshStatePulling) {// 即将刷新 && 手松开
        	// 开始刷新
        	[self beginRefreshing];
    	} else if (pullingPercent < 1) {
        	self.pullingPercent = pullingPercent;
    	}
	}
上面的其实就是根据state的不同，进行相应的操作，然后更改state，在每一次更改state的时候，就发生了哪些变化呢，我们看看下面的方法

	MJRefreshHeader文件
	- (void)setState:(MJRefreshState)state
	{
   	 MJRefreshCheckState
    
    	// 根据状态做事情
    	if (state == MJRefreshStateIdle) {
        	if (oldState != MJRefreshStateRefreshing) return;
        
        	// 保存刷新时间
        	[[NSUserDefaults standardUserDefaults] setObject:[NSDate date] forKey:self.lastUpdatedTimeKey];
        	[[NSUserDefaults standardUserDefaults] synchronize];
        
        	// 恢复inset和offset
        	[UIView animateWithDuration:MJRefreshSlowAnimationDuration animations:^{
            	self.scrollView.mj_insetT += self.insetTDelta;
            
            	// 自动调整透明度
            	if (self.isAutomaticallyChangeAlpha) self.alpha = 0.0;
        } completion:^(BOOL finished) {
            	self.pullingPercent = 0.0;
            
            	if (self.endRefreshingCompletionBlock) {
                	self.endRefreshingCompletionBlock();
            	}
        	}];
    	} else if (state == MJRefreshStateRefreshing) {
         	dispatch_async(dispatch_get_main_queue(), ^{
            	[UIView animateWithDuration:MJRefreshFastAnimationDuration animations:^{
                	CGFloat top = self.scrollViewOriginalInset.top + self.mj_h;
                	// 增加滚动区域top
                	self.scrollView.mj_insetT = top;
                	// 设置滚动位置
                	[self.scrollView setContentOffset:CGPointMake(0, -top) animated:NO];
            } completion:^(BOOL finished) {
                	[self executeRefreshingCallback];
            	}];
         	});
    	}
	}

执行setState方法的时候，进行了界面的操作。