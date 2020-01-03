---
layout:     post
title:      UITableView性能优化
subtitle:   
date:       2019-12-25
author:     夏天无泪
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - 学习整理
---

## UITableView性能优化  

>卡顿产生的原因：因为CPU或者GPU所花费的时间过长，导致垂直信号来的时候，CPU计算或者GPU渲染未完成，从而掉帧。  

### 一、减轻CPU负荷
 
#### 1、提前计算好cell的高度,缓存在相应的数据源模型中  

>我们知道CPU的主要负责快速调度任务,大量计算工作,所以在tableView快速滚动的过程中让CPU的计算量降低是优化应该考虑的方向.下面总结了三个方面来尽可能的降低CPU计算:   

我们知道tableView的代理回调方法中,先调用的是返回cell高度的方法,然后在返回实例化cell的方法.我们可以在返回cell高度时,提前计算好cell的高度,缓存到数据源模型中。  

**实例代码如下:**  

**viewController.m**  

```  
// 生成模型数据的时候计算视图的高度
- (NSArray *)getRandomData {
    NSMutableArray *models = [NSMutableArray array];
    int number = arc4random_uniform(30);
    for (int i = 0; i < 20 + number; i++) {
        NewsModel *model = [[NewsModel alloc] init];
        ......
        model.rowHeight = [self calculateNewsCellHeight:model];
        [models addObject:model];
    }
    return models.copy;
}

/// 计算高度
- (CGFloat)calculateNewsCellHeight:(NewsModel *)model {
    float contentHeight = 64;
    contentHeight += ([self getContentHeight:model.content] + 10);
    if (model.imgs.count > 0) {
        contentHeight += (kImgViewWH + 10);
    }
    return (contentHeight + 44 + 5);    // 64,10,10,44,5等都是固定高度
}

- (CGFloat)getContentHeight:(NSString *)content {
    CGSize size = [content boundingRectWithSize:CGSizeMake(kScreenWidth - 20, MAXFLOAT)
                                        options:NSStringDrawingTruncatesLastVisibleLine|NSStringDrawingUsesLineFragmentOrigin|NSStringDrawingUsesFontLeading
                                       attributes:@{NSFontAttributeName:[UIFont systemFontOfSize:16 ]}
                                          context:nil].size;
    return size.height;
}

#pragma mark - updateData

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    NewsModel *model = [self.dataSource objectAtIndex:indexPath.row];
    CalculateNewsCell *cell = [tableView dequeueReusableCellWithIdentifier:cellId];
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    cell.model = model;
    return cell;
}

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    NewsModel *model = [self.dataSource objectAtIndex:indexPath.row];
    return model.rowHeight;
}
```  

**CalculateNewsCell.m**  

```
- (void)setModel:(NewsModel *)model {
    _model = model;
   
    self.contentLbe.text = model.content;
    [self.contentLbe fitSizeHeight];  // 不能使用sizeToFit
}
```  

**UILabel (Extension)**  

```  
- (void)fitSizeHeight {
    [self fitSizeHeight:0];
}

- (void)fitSizeHeight:(float)padding {
    float srcWidth = self.width;
    [self sizeToFit];
    float srcHeight = self.height;
    self.width = srcWidth;
    self.height = srcHeight + padding * 2;
}
```  

>注意事项:
1.UITableViewCell视图中给UILabel赋值完 text后，一定要调用fitSizeHeight方法，不能调用系统的sizeToFit方法，要不然高度会计算错误。  

#### 2、尽可能的减少storyboard,xib的使用  

通过Interface知道xib或者storyboard本身就是一个xml文件，添加删除控件必然中间多了一个`encode/decode`过程，增加了cpu的计算量。并且还要避免臃肿的 XIB 文件，因为XIB文件在主线程中进行加载布局。当用到一些自定义View或者XIB文件时，XIB的加载会把所有内容加载进来，如果XIB里面的一些控件并不会用到，这就可能造成一些资源的消耗浪费。

比如使用纯代码布局，使用masonry约束布局或者手动计算布局。本人就是完全抛弃了storyboard和xib。
  
** 示例代码；**  

* **CalculateNewsCell.h**  

``` 
@class NewsModel;
/// 缓存 cell 的高度
@interface CalculateNewsCell : UITableViewCell
/** model */
@property(nonatomic, strong)NewsModel *model;
@end
```   

* **CalculateNewsCell.m**  

```
@interface CalculateNewsCell()
/** icon */
@property(nonatomic, strong)UIImageView *iconImgView;
......
@end

@implementation CalculateNewsCell

#define kImgViewWH (kScreenWidth - 20 - 15) / 4.0

- (instancetype)initWithStyle:(UITableViewCellStyle)style reuseIdentifier:(NSString *)reuseIdentifier {
    self = [super initWithStyle:style reuseIdentifier:reuseIdentifier];
    if (self) {
        self.contentView.backgroundColor = [UIColor whiteColor];
        [self drawUI];
    }
    return self;
}

#pragma mark - drawUI

- (void)drawUI {
    self.contentView.width = kScreenWidth;
    self.iconImgView.y = 10;
    self.iconImgView.x = 10;
    [self.contentView addSubview:self.iconImgView];
      
    ......
}

#pragma mark - set

- (void)setModel:(NewsModel *)model {
    _model = model;
    [self.iconImgView sd_setImageWithURL:[NSURL URLWithString:model.icon]];
    ......
}

#pragma mark - lazy

- (UIImageView *)iconImgView {
    if (_iconImgView == nil) {
        _iconImgView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, 44, 44)];
        _iconImgView.layer.cornerRadius = 22;
        _iconImgView.layer.masksToBounds = YES;
    }
    return _iconImgView;
}
``` 

#### 3、滑动过程中尽量减少重新布局  

自动布局就是给控件添加约束，约束最终还是转换成frame。所以在满足业务需求情况下,如果图层层次较为复杂，要尽量减少自动布局约束，转为手动计算布局，大量的约束重叠也会增加cpu的计算量。

比如，如果内容相对固定，可以将UILabel的宽高写死，只是更新其文本内容即可

代码：  

```
// 避免重复计算,宽高写死
UILabel *titleLbe = [[UILabel alloc] initWithFrame:CGRectMake(0, 0, kScreenWidth - 200, 20)];

- (void)setModel:(NewsModel *)model {
    _model = model;
    ......
    // 更新文本内容即可
    self.titleLbe.text = model.title;
}
```  

> 本示例中就是将头像尺寸，标题，副标题，最下面的分享按钮，评论按钮，点赞按钮等视图尺寸固定，只是更新内容即可，避免了 CPU 的计算。


### 二、 按需加载  

**当滑动 UITableView 时，按需加载对应的内容**

```
#pragma mark - UIScrollViewDelegate

// 按需加载 - 如果目标行与当前行相差超过指定行数，只在目标滚动范围的前后指定3行加载。
- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset {
    NSIndexPath *ip = [self.tableView indexPathForRowAtPoint:CGPointMake(0, targetContentOffset->y)];   // 停止拖拽后,预计滑动停止后到的偏移量
    NSIndexPath *cip = [[self.tableView indexPathsForVisibleRows] firstObject]; // 当前可视区域内 cell 组
    NSInteger skipCount = 8;
    NSLog(@"targetContentOffset = %f",targetContentOffset->y);
    NSLog(@"indexPathForRowAtPoint = %@",ip);
    NSLog(@"visibleRows = %@",[self.tableView indexPathsForVisibleRows]);
    if (labs(cip.row - ip.row) > skipCount) {   // labs-返回 x 的绝对值,进入该方法,说明滑动太厉害了,预计停留位置与当前可视区域范围内差 8 个cell 以上了.
        // 拖拽停止滑动停止后,即将显示的 cell 索引组
        NSArray *temp = [self.tableView indexPathsForRowsInRect:CGRectMake(0, targetContentOffset->y, self.tableView.width, self.tableView.height)];
        NSMutableArray *arrM = [NSMutableArray arrayWithArray:temp];
        NSLog(@"temp = %@",temp);
        if (velocity.y < 0) {   // 向上滑动-即加载更多数据
            NSIndexPath *indexPath = [temp lastObject];
            if (indexPath.row + 3 < self.dataSource.count) {    // 滑动停止后出现的 cell 索引仍在数据源范围之内
                [arrM addObject:[NSIndexPath indexPathForRow:indexPath.row + 1 inSection:0]];
                [arrM addObject:[NSIndexPath indexPathForRow:indexPath.row + 2 inSection:0]];
                [arrM addObject:[NSIndexPath indexPathForRow:indexPath.row + 3 inSection:0]];
            }
        } else {    // 向下滑动-加载之前的数据
            NSIndexPath *indexPath = [temp firstObject];
            if (indexPath.row > 3) {
                [arrM addObject:[NSIndexPath indexPathForRow:indexPath.row - 3 inSection:0]];
                [arrM addObject:[NSIndexPath indexPathForRow:indexPath.row - 2 inSection:0]];
                [arrM addObject:[NSIndexPath indexPathForRow:indexPath.row - 1 inSection:0]];
            }
        }
        
        [self.needLoadArray addObjectsFromArray:arrM];
    }
}
```  

https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/UIVivew渲染和卡顿/0DF84BB1D897D9C0C6EA5D9DF8B3B14E.png?raw=true


**UITableViewDataSource 数据源**  

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    NeedLoadNewsCell *cell = [tableView dequeueReusableCellWithIdentifier:cellId];
    [self drawCell:cell withIndexPath:indexPath];
    cell.delegate = self;   // VC作为Cell视图的代理对象
    return cell;
}

// 按需绘制
- (void)drawCell:(NeedLoadNewsCell *)cell withIndexPath:(NSIndexPath *)indexPath{
    NewsModel *model = [self.dataSource objectAtIndex:indexPath.row];
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    [cell clear];
    cell.model = model;
    // 如果当前 cell 不在需要绘制的cell 中,则直接 pass
    if (self.needLoadArray.count > 0 && [self.needLoadArray indexOfObject:indexPath] == NSNotFound) {
        [cell clear];
        return;
    }
    if (_scrollToToping) {
        return;
    }
    [cell draw];
}
```  

* **一键返回至顶部方法判断**  

```
// 开始滚动到顶部
- (BOOL)scrollViewShouldScrollToTop:(UIScrollView *)scrollView {
    _scrollToToping = YES;
    return YES;
}

- (void)scrollViewDidEndScrollingAnimation:(UIScrollView *)scrollView {
    _scrollToToping = NO;
    [self loadContent];
}

// 已经滚动到顶部
- (void)scrollViewDidScrollToTop:(UIScrollView *)scrollView {
    _scrollToToping = NO;
    [self loadContent];
}

- (void)loadContent {
    if (_scrollToToping) {
        return;
    }
    if (self.tableView.indexPathsForVisibleRows.count <= 0) {
        return;
    }
    if (self.tableView.visibleCells && self.tableView.visibleCells.count > 0) {
        for (NeedLoadNewsCell *cell in [self.tableView.visibleCells copy]) {
            [cell draw];
        }
    }
}
```  

### 三、 异步绘制  

当视图层级比较多的时候，可以采用异步绘制的方式，通过UIGraphics将内容绘制然后生成一张图片进行展示。  

#### 1.处理数据源  

当我们请求到了数据后，需要根据后端返回的数据，根据内容将布局计算出来，后面再进行绘制，部分代码示例如下  

```  
// 计算 frame
// 1.内容 + 图片预览
{
    NSString *content = [NSString stringWithFormat:@"%@: %@ %@",model.specialWord,model.content,model.link];
    float width = kScreenWidth - SIZE_GAP_LEFT * 2;
    CGSize size =  [content sizeWithConstrainedToWidth:width fromFont:FontWithSize(SIZE_FONT_SUBCONTENT) lineSpace:5];
    NSInteger sizeHeight = size.height + .5;
    model.textFrame = CGRectMake(SIZE_GAP_LEFT, SIZE_GAP_BIG + 64, width, sizeHeight);
    sizeHeight += SIZE_GAP_BIG * 2;
    
    if (model.imgs.count > 0) { // 图片
        sizeHeight += (SIZE_GAP_IMG + SIZE_IMAGE + SIZE_GAP_IMG);
    }
    sizeHeight += SIZE_GAP_BIG;
    model.contentFrame = CGRectMake(0, 64, kScreenWidth, sizeHeight);
}  
```  

* `sizeWithConstrainedToWidth:(float)width fromFont:(UIFont *)font1 lineSpace:(float)lineSpace`  

```  
- (CGSize)sizeWithConstrainedToWidth:(float)width fromFont:(UIFont *)font1 lineSpace:(float)lineSpace{
    return [self sizeWithConstrainedToSize:CGSizeMake(width, CGFLOAT_MAX) fromFont:font1 lineSpace:lineSpace];
}

- (CGSize)sizeWithConstrainedToSize:(CGSize)size fromFont:(UIFont *)font1 lineSpace:(float)lineSpace{
    CGFloat minimumLineHeight = font1.pointSize,maximumLineHeight = minimumLineHeight, linespace = lineSpace;
    CTFontRef font = CTFontCreateWithName((__bridge CFStringRef)font1.fontName,font1.pointSize,NULL);
    CTLineBreakMode lineBreakMode = kCTLineBreakByWordWrapping;
    //Apply paragraph settings
    CTTextAlignment alignment = kCTLeftTextAlignment;
    CTParagraphStyleRef style = CTParagraphStyleCreate((CTParagraphStyleSetting[6]){
        {kCTParagraphStyleSpecifierAlignment, sizeof(alignment), &alignment},
        {kCTParagraphStyleSpecifierMinimumLineHeight,sizeof(minimumLineHeight),&minimumLineHeight},
        {kCTParagraphStyleSpecifierMaximumLineHeight,sizeof(maximumLineHeight),&maximumLineHeight},
        {kCTParagraphStyleSpecifierMaximumLineSpacing, sizeof(linespace), &linespace},
        {kCTParagraphStyleSpecifierMinimumLineSpacing, sizeof(linespace), &linespace},
        {kCTParagraphStyleSpecifierLineBreakMode,sizeof(CTLineBreakMode),&lineBreakMode}
    },6);
    NSDictionary* attributes = [NSDictionary dictionaryWithObjectsAndKeys:(__bridge id)font,(NSString*)kCTFontAttributeName,(__bridge id)style,(NSString*)kCTParagraphStyleAttributeName,nil];
    NSMutableAttributedString *string = [[NSMutableAttributedString alloc] initWithString:self attributes:attributes];
    //    [self clearEmoji:string start:0 font:font1];
    CFAttributedStringRef attributedString = (__bridge CFAttributedStringRef)string;
    CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attributedString);
    CGSize result = CTFramesetterSuggestFrameSizeWithConstraints(framesetter, CFRangeMake(0, [string length]), NULL, size, NULL);
    CFRelease(framesetter);
    CFRelease(font);
    CFRelease(style);
    string = nil;
    attributes = nil;
    return result;
}  
```  

#### 2.异步绘制数据  

在赋值数据模型里面进行内容的绘制  

```
#pragma mark - set

- (void)setModel:(NewsModel *)model {
    _model = model;
    
    [self.iconImgView sd_setImageWithURL:[NSURL URLWithString:model.icon]];
    
    if (_drawed) {
        return;
    }
    NSUInteger flag = _drawColorFlag;
    _drawed = YES;
    
    // 开始异步绘制
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        // 先开启一个上下文
        UIGraphicsBeginImageContextWithOptions(model.totalFrame.size, YES, 0);
        CGContextRef context = UIGraphicsGetCurrentContext();
        
        // 整个 cell区域
        [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];
        CGContextFillRect(context, model.totalFrame);
        
        // 先绘制内容的背景视图
        [[UIColor colorWithRed:243/255.0 green:243/255.0 blue:243/255.0 alpha:1] set];
        CGContextFillRect(context, model.contentFrame);
        
        // 内容视图上方分割线
        [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set];
        CGContextFillRect(context, CGRectMake(0, model.contentFrame.origin.y, model.contentFrame.size.width, 0.5));
        
        // title + subTitle
        {
            // title
            float leftX = SIZE_GAP_LEFT + SIZE_AVATAR + SIZE_GAP_BIG;
            float x = leftX;
            float y = (SIZE_AVATAR - (SIZE_FONT_NAME + SIZE_FONT_SUBTITLE + 6)) * 0.5 - 2 + SIZE_GAP_TOP + SIZE_GAP_SMALL - 5;
            [model.title drawInContext:context
                           withPosition:CGPointMake(x, y)
                                andFont:FontWithSize(SIZE_FONT_NAME)
                           andTextColor:[UIColor colorWithRed:106/255.0 green:140/255.0 blue:181/255.0 alpha:1]
                              andHeight:model.totalFrame.size.height];
            
            // subTitle
            y += (SIZE_FONT_NAME + 5);
            float fromX = leftX;
            float titleWidth = kScreenWidth - leftX;
            [model.subTitle drawInContext:context
                              withPosition:CGPointMake(fromX, y)
                                   andFont:FontWithSize(SIZE_FONT_SUBTITLE)
                              andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1]
                                 andHeight:model.totalFrame.size.height
                                  andWidth:titleWidth];
        }
        
        // 点赞+评论按钮 - 顶部分割线
        [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set];
        CGContextFillRect(context, CGRectMake(0, model.contentFrame.origin.y + model.contentFrame.size.height, kScreenWidth, 0.5));
        
        // 点赞+评论按钮 - 底部分割线
        [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set];
        CGContextFillRect(context, CGRectMake(0, model.contentFrame.origin.y + model.contentFrame.size.height + 43, kScreenWidth, 0.5));
        
        // 生成图片
        UIImage *temp = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        dispatch_async(dispatch_get_main_queue(), ^{
            if (flag == _drawColorFlag) {
                self.contentImgView.frame = model.totalFrame;
                self.contentImgView.image = nil;
                self.contentImgView.image = temp;
            }
        });
    });
    
    [self drawText];
    [self loadThumb];
    [self setData];
}  
```  

}
其中我们用了一个大神封装好的` VVeboLabel`，详细源码项目链接中有,有需要自行查阅。  

```  
- (void)drawText {
    if (label == nil) {
        [self addLabel];
    }
    label.frame = _model.textFrame;
    [label setText:[NSString stringWithFormat:@"%@: %@ %@",_model.specialWord,_model.content,_model.link]];
}  
```  

在调用`setModel:`赋值数据之前，我们需要做清除数据的操作  

```  
/// 清空视图
- (void)clear {
    if (!_drawed) {
        return;
    }
    self.contentImgView.frame = CGRectZero;
    self.contentImgView.image = nil;
    
    [label clear];
    // 清除图片
    for (UIImageView *imgView in self.imgListView.subviews) {
        [imgView sd_cancelCurrentAnimationImagesLoad];
    }
    self.imgListView.hidden = YES;
    _drawColorFlag = arc4random();
    _drawed = NO;
}  
```   

### 四.延时加载图片  

Runloop每次循环都会对你界面上的UI绘制一遍，主要是速度快，我们看不出来，当界面中出现高清的图片时因为绘制的慢，就会导致卡顿。 所以我们监听runloop的状态，每次即将休眠的时候，即处于kCFRunLoopBeforeWaiting状态时才去绘制加载图片。

**核心代码如下：**  

* 监听 runloop 状态并且执行当 runloop 处于kCFRunLoopBeforeWaiting时需要执行的代码块  

```   
// 监听 runloop 状态
- (void)addRunloopObserver {
    // 获取当前 runloop
    //获得当前线程的runloop，因为我们现在操作都是在主线程，这个方法就是得到主线程的runloop
    CFRunLoopRef runloop = CFRunLoopGetCurrent();

    //定义一个观察者,这是一个结构体
    CFRunLoopObserverContext context = {
        0,
        (__bridge void *)(self),
        &CFRetain,
        &CFRelease,
        NULL
    };

    // 定义一个观察者
    static CFRunLoopObserverRef defaultModeObsever;
    // 创建观察者
    defaultModeObsever = CFRunLoopObserverCreate(NULL,
                                                 kCFRunLoopBeforeWaiting,   // 观察runloop等待的时候就是处于NSDefaultRunLoopMode模式的时候
                                                 YES,   // 是否重复观察
                                                 NSIntegerMax - 999,
                                                 &Callback, // 回掉方法，就是处于NSDefaultRunLoopMode时候要执行的方法
                                                 &context);
    
    // 添加当前 RunLoop 的观察者
    CFRunLoopAddObserver(runloop, defaultModeObsever, kCFRunLoopDefaultMode);
    //c语言有creat 就需要release
    CFRelease(defaultModeObsever);
}

// 每次 runloop 回调执行代码块
static void Callback(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    DelayLoadImgViewController *vc = (__bridge DelayLoadImgViewController *)(info);  // 这个info就是我们在context里面放的self参数
    
    if (vc.tasks.count == 0) {
        return;
    }
    
    BOOL result = NO;
    while (result == NO && vc.tasks.count) {
        NSLog(@"开始执行加载图片总任务数:%d",vc.tasks.count);
        // 取出任务
        RunloopBlock unit = vc.tasks.firstObject;
        // 执行任务
        result = unit();
        // d干掉第一个任务
        [vc.tasks removeObjectAtIndex:0];
    }
    
```  
   
 * 将加载绘制图片丢到任务中  

 ```
 - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    NewsModel *model = [self.dataSource objectAtIndex:indexPath.row];
    DelayLoadImgCell *cell = [tableView dequeueReusableCellWithIdentifier:cellId];
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    cell.model = model;
    // 将加载绘制图片操作丢到任务中去
    [self addTask:^BOOL{
        [cell drawImg];
        return YES;
    }];
    return cell;
}
```  

* UITableViewCell 中绘制图片的方法  

```  
/// 绘制图片
- (void)drawImg {
    if (_model.imgs.count > 0) {
        __block float posX = 0;
        [_model.imgs enumerateObjectsUsingBlock:^(NSString *obj, NSUInteger idx, BOOL *stop) {
            UIImageView *imgView = [[UIImageView alloc] initWithFrame:CGRectMake(posX, 0, kImgViewWH, kImgViewWH)];
            [imgView sd_setImageWithURL:[NSURL URLWithString:obj]];
            imgView.layer.cornerRadius = 5;
            imgView.layer.masksToBounds = YES;
            
            [self.imgListView addSubview:imgView];
            posX += (5 + kImgViewWH);
            if (idx >= 3) {
                *stop = YES;
            }
        }];
    }
}
``` 

https://github.com/xiatianwulei/xiatianwulei.github.io/blob/master/img/media/UIVivew渲染和卡顿/1E55FC2FAA911853B96265E700BDBD35.png?raw=true  

>每次当我们滑动时，图片不显示，因为这个时候runloop不处于kCFRunLoopBeforeWaiting状态。当我们停止拖拽滑动时，runloop 处于kCFRunLoopBeforeWaiting状态，然后加载绘制图片。
  
借鉴的开源项目[VVeboTableViewDemo](https://github.com/johnil/VVeboTableViewDemo)  


