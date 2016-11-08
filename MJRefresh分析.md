##MJRefresh分析

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
	
作者利用runtime的技巧给这个分类添加了五个属性和一个方法，然后封装好的刷新控件添加给UIScrollview   
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