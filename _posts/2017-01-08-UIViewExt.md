---
layout:     post
title:      UIViewExt
subtitle:   
date:       2017-01-07
author:     夏天无泪
header-img: img/post-bg-iWatch.jpg
catalog: true
tags:
    - SwiftExt
---


```
**通过UIView对象找到其所在的UIViewController**
extension UIView {
    //返回该view所在VC
    func firstViewController() -> UIViewController? {
        for view in sequence(first: self.superview, next: { $0?.superview }) {
            if let responder = view?.next {
                if responder.isKind(of: UIViewController.self){
                    return responder as? UIViewController
                }
            }
        }
        return nil
    }
}
```