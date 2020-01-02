---
layout:     post
title:      UITableView性能优化
subtitle:   
date:       2019-12-25
author:     夏天无泪
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - 学习整理
---

## UITableView性能优化  

>我们知道CPU的主要负责快速调度任务,大量计算工作,所以在tableView快速滚动的过程中让CPU的计算量降低是优化应该考虑的方向.下面总结了三个方面来尽可能的降低CPU计算:  
 
### 1、提前计算好cell的高度,缓存在相应的数据源模型中  

我们知道tableView的代理回调方法中,先调用的是返回cell高度的方法,然后在返回实例化cell的方法.我们可以在返回cell高度时,提前计算好cell的高度,缓存到数据源模型中。  

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

### 2、尽可能的减少storyboard,xib的使用  

通过Interface知道xib或者storyboard本身就是一个xml文件，添加删除控件必然中间多了一个encode/decode过程，增加了cpu的计算量。并且还要避免臃肿的 XIB 文件，因为XIB文件在主线程中进行加载布局。当用到一些自定义View或者XIB文件时，XIB的加载会把所有内容加载进来，如果XIB里面的一些控件并不会用到，这就可能造成一些资源的消耗浪费。

比如使用纯代码布局，使用masonry约束布局或者手动计算布局。本人就是完全抛弃了storyboard和xib。
  
**  CalculateNewsCell.h**  

``` 
@class NewsModel;
/// 缓存 cell 的高度
@interface CalculateNewsCell : UITableViewCell
/** model */
@property(nonatomic, strong)NewsModel *model;
@end
```   

**CalculateNewsCell.m**  

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

### 3、滑动过程中尽量减少重新布局  

自动布局就是给控件添加约束，约束最终还是转换成frame。所以在满足业务需求情况下,如果图层层次较为复杂，要尽量减少自动布局约束，转为手动计算布局，大量的约束重叠也会增加cpu的计算量。

比如，如果内容相对固定，可以将UILabel的宽高写死，只是更新其文本内容即可


### 4、 按需加载  

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

**一键返回至顶部方法判断**  

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

### 没总结完

